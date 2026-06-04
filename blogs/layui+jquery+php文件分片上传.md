# 使用LayUI+jQuery+PHP实现大文件分片上传

### 背景

​	帮业务解决问题时发现某一个上传文件的功能可能因为文件过大导致POST数据丢失的问题。排查之后发现是因为`php`配置的`upload_max_filesize`和`post_max_size`都为20M，而业务上传的文件大小为27.3M，虽然请求可以正常到达后端，但是却没有POST数据。为了快速解决问题，找运维改了`php`的配置，功能恢复正常。

​	然而这并不是长久之计，倘若文件大小为100M、1G，又该如何解决。因此，我便想要使用分片上传来实现对大文件的上传功能，并且为了方便在其他功能中使用，需要将其设计为即开即用的模块化工具类。

### 效果

![image-20250327180214018](./../../../Code/FBA-DOC/images/layui+jquery+php文件分片上传/image-20250327180214018.png)

![image-20250327180303802](./../../../Code/FBA-DOC/images/layui+jquery+php文件分片上传/image-20250327180303802.png)

### 实现

核心功能其实就两部分：前端对文件的分片和上传以及后端对分片的接收合并。下面是源码，供大家参考指正。

- 前端`ChunkUploader`类

  ```js
  class ChunkUploader {
      constructor(options = {}) {
          this.chunkSize = options.chunkSize || 2 * 1024 * 1024;
          this.initUrl = options.initUrl || '/index.php?mod=common&act=initUpload';
          this.uploadUrl = options.uploadUrl || '/index.php?mod=common&act=uploadChunk';
          this.completeUrl = options.completeUrl || '/index.php?mod=common&act=completeUpload';
          this.onProgress = options.onProgress || (() => {});
          this.onSuccess = options.onSuccess || (() => {});
          this.onError = options.onError || (() => {});
          
          // 新增属性
          this.isPaused = false;
          this.isUploading = false;
          this.uploadedChunks = 0;
          this.totalChunks = 0;
          this.currentFile = null;
          this.uploadId = null;
      }
  
      async upload(file) {
          try {
              this.currentFile = file;
              this.isUploading = true;
              this.isPaused = false;
              
              // 初始化上传
              const initResult = await $.ajax({
                  url: this.initUrl,
                  type: 'POST',
                  dataType: 'json',
                  data: {
                      fileName: file.name,
                      fileSize: file.size
                  }
              });
              if (initResult.errCode !== 200) {
                  throw new Error(initResult.errMsg || '初始化上传失败');
              }
              
              const { uploadId, chunkSize } = initResult.data;
              this.uploadId = uploadId;
              
              // 计算分片
              const actualChunkSize = chunkSize || this.chunkSize;
              this.totalChunks = Math.ceil(file.size / actualChunkSize);
              this.uploadedChunks = 0;
  
              // 上传分片
              await this.uploadChunks(file, uploadId, actualChunkSize);
              
              // 如果没有暂停且所有分片已上传，则完成上传
              if (!this.isPaused && this.uploadedChunks === this.totalChunks) {
                  const completeResult = await $.ajax({
                      url: this.completeUrl,
                      type: 'POST',
                      dataType: 'json',
                      data: {
                          uploadId: uploadId,
                          fileName: file.name
                      }
                  });
                  
                  if (completeResult.errCode !== 200) {
                      throw new Error(completeResult.errMsg || '完成上传失败');
                  }
                  
                  this.isUploading = false;
                  this.onSuccess(completeResult);
                  return completeResult;
              }
              
              return { code: 100, msg: '上传暂停' };
          } catch(e) {
              this.isUploading = false;
              this.onError(e);
              throw e;
          }
      }
      
      async uploadChunks(file, uploadId, chunkSize) {
          for(let i = this.uploadedChunks; i < this.totalChunks; i++) {
              // 如果暂停，则中断上传
              if (this.isPaused) {
                  break;
              }
              
              const chunk = file.slice(i * chunkSize, Math.min((i + 1) * chunkSize, file.size));
              const formData = new FormData();
              formData.append('chunk', chunk);
              formData.append('chunkIndex', i);
              formData.append('uploadId', uploadId);
  
              const result = await $.ajax({
                  url: this.uploadUrl,
                  type: 'POST',
                  dataType: 'json',
                  data: formData,
                  processData: false,
                  contentType: false
              });
              
              if (result.errCode !== 200) {
                  throw new Error(result.errMsg || '分片上传失败');
              }
  
              this.uploadedChunks++;
              this.onProgress(Math.floor(this.uploadedChunks * 100 / this.totalChunks));
          }
      }
      
      // 暂停上传
      pause() {
          if (this.isUploading) {
              this.isPaused = true;
          }
      }
      
      // 恢复上传
      async resume() {
          if (this.isPaused && this.currentFile && this.uploadId) {
              this.isPaused = false;
              
              // 继续上传剩余分片
              await this.uploadChunks(
                  this.currentFile, 
                  this.uploadId, 
                  this.chunkSize
              );
              
              // 如果所有分片已上传，则完成上传
              if (!this.isPaused && this.uploadedChunks === this.totalChunks) {
                  const completeResult = await $.ajax({
                      url: this.completeUrl,
                      type: 'POST',
                      dataType: 'json',
                      data: {
                          uploadId: this.uploadId,
                          fileName: this.currentFile.name
                      }
                  });
                  
                  if (completeResult.errCode !== 200) {
                      throw new Error(completeResult.errMsg || '完成上传失败');
                  }
                  
                  this.isUploading = false;
                  this.onSuccess(completeResult);
                  return completeResult;
              }
          }
      }
  } 
  ```

- 后端`chunkUploadService`类

  ```php
  <?php
  /**
   * 分片上传服务
   */
  class chunkUploadService extends CommonService {
      /**
       * 临时存储目录
       */
      private $tempDir;
      
      /**
       * 构造函数
       */
      public function __construct() {
          $this->tempDir = WEB_PATH . 'html/upload/temp/';
          
          // 确保临时目录存在
          if (!file_exists($this->tempDir)) {
              mkdir($this->tempDir, 0777, true);
          }
      }
      
      /**
       * 初始化上传
       * @param array $params 参数
       * @return array 结果
       */
      public function initUpload($params) {
          try {
              $fileName = isset($params['fileName']) ? $params['fileName'] : '';
              $fileSize = isset($params['fileSize']) ? intval($params['fileSize']) : 0;
              
              if (empty($fileName) || $fileSize <= 0) {
                  throw new Exception('参数错误');
              }
              
              // 生成上传ID
              $uploadId = md5($fileName . $fileSize . microtime(true));
              
              // 创建上传记录
              $uploadDir = $this->tempDir . $uploadId . '/';
              if (!file_exists($uploadDir)) {
                  mkdir($uploadDir, 0777, true);
              }
              
              // 保存上传信息
              $uploadInfo = array(
                  'fileName' => $fileName,
                  'fileSize' => $fileSize,
                  'uploadId' => $uploadId,
                  'chunkSize' => 2 * 1024 * 1024, // 2MB
                  'chunks' => array(),
                  'createTime' => date('Y-m-d H:i:s')
              );
              
              file_put_contents($uploadDir . 'info.json', json_encode($uploadInfo));
              return ['uploadId' => $uploadId,'chunkSize' => $uploadInfo['chunkSize']];
          } catch (Exception $e) {
              $message = $e->getMessage();
              $code    = $e->getCode();
              return $this->returnError($message, $code);
          }
      }
      
      /**
       * 上传分片
       * @param string $uploadId 上传ID
       * @param int $chunkIndex 分片索引
       * @param array $file 文件信息
       * @return array 结果
       */
      public function uploadChunk($uploadId, $chunkIndex, $file) {
          try {
              if (empty($uploadId) || !isset($file['tmp_name'])) {
                  throw new Exception('参数错误');
              }
          
              $uploadDir = $this->tempDir . $uploadId . '/';
              if (!file_exists($uploadDir)) {
                  throw new Exception('上传ID无效');
              }
              
              // 读取上传信息
              $infoFile = $uploadDir . 'info.json';
              if (!file_exists($infoFile)) {
                  throw new Exception('上传信息不存在');
              }
              
              $uploadInfo = json_decode(file_get_contents($infoFile), true);
              
              // 保存分片
              $chunkFile = $uploadDir . 'chunk_' . $chunkIndex;
              if (!move_uploaded_file($file['tmp_name'], $chunkFile)) {
                  throw new Exception('保存分片失败');
              }
              
              // 更新上传信息
              if (!in_array($chunkIndex, $uploadInfo['chunks'])) {
                  $uploadInfo['chunks'][] = $chunkIndex;
              }
              file_put_contents($infoFile, json_encode($uploadInfo));
              return ['chunkIndex' => $chunkIndex];
          } catch (Exception $e) {
              $message = $e->getMessage();
              $code    = $e->getCode();
              return $this->returnError($message, $code);
          }
      }
      
      /**
       * 完成上传
       * @param string $uploadId 上传ID
       * @param string $fileName 文件名
       * @param string $targetPath 目标路径
       * @return array 结果
       */
      public function completeUpload($uploadId, $fileName, $targetPath) {
          try {
              if (empty($uploadId) || empty($fileName)) {
                  throw new Exception('参数错误');
              }
              
              $uploadDir = $this->tempDir . $uploadId . '/';
              if (!file_exists($uploadDir)) {
                  throw new Exception('上传ID无效');
              }
              
              // 读取上传信息
              $infoFile = $uploadDir . 'info.json';
              if (!file_exists($infoFile)) {
                  throw new Exception('上传信息不存在');
              }
              
              $uploadInfo = json_decode(file_get_contents($infoFile), true);
              
              // 计算总分片数
              $totalChunks = ceil($uploadInfo['fileSize'] / $uploadInfo['chunkSize']);
              
              // 检查分片是否完整
              if (count($uploadInfo['chunks']) < $totalChunks) {
                  throw new Exception('分片不完整');
              }
              
              // 确保目标目录存在
              $targetDir = dirname($targetPath);
              if (!file_exists($targetDir)) {
                  mkdir($targetDir, 0777, true);
              }
              
              // 合并分片
              $targetFile = fopen($targetPath, 'wb');
              if (!$targetFile) {
                  throw new Exception('无法创建目标文件');
              }
              
              for ($i = 0; $i < $totalChunks; $i++) {
                  $chunkFile = $uploadDir . 'chunk_' . $i;
                  if (!file_exists($chunkFile)) {
                      fclose($targetFile);
                      throw new Exception('分片 ' . $i . ' 不存在');
                  }
                  
                  $chunk = file_get_contents($chunkFile);
                  fwrite($targetFile, $chunk);
              }
              
              fclose($targetFile);
              
              // 清理临时文件
              $this->cleanupTempFiles($uploadDir);
              
              return ['path' => str_replace(WEB_PATH, '', $targetPath)];
          } catch (Exception $e) {
              $message = $e->getMessage();
              $code    = $e->getCode();
              return $this->returnError($message, $code);
          }
      }
      
      /**
       * 清理临时文件
       * @param string $dir 目录
       */
      private function cleanupTempFiles($dir) {
          if (is_dir($dir)) {
              $objects = scandir($dir);
              foreach ($objects as $object) {
                  if ($object != "." && $object != "..") {
                      if (is_dir($dir . "/" . $object)) {
                          $this->cleanupTempFiles($dir . "/" . $object);
                      } else {
                          unlink($dir . "/" . $object);
                      }
                  }
              }
              rmdir($dir);
          }
      }
  }
  ```

### 使用方法

- 前端处理：调用分片上传工具类

  ```js
  // 创建自定义加载层，包含进度显示
  var loadingHtml = '<div style="padding: 20px; line-height: 22px; text-align: center;">' +
    '<i class="layui-icon layui-icon-loading layui-anim layui-anim-rotate layui-anim-loop" style="font-size: 30px;"></i>' +
    '<div id="uploadProgressText" style="margin-top: 15px;">文件上传中 (0%)</div>' +
    '</div>';
  
  var loadIndex2 = layer.open({
    type: 1,
    title: false,
    closeBtn: 0,
    shade: [0.3, '#fff'],
    area: ['300px', '100px'],
    content: loadingHtml
  });
  
  // 创建上传实例
  uploader = new ChunkUploader({
    initUrl: 'json.php?mod=thirdWH&act=initUploadAction&jsonp=1',
    uploadUrl: 'json.php?mod=thirdWH&act=uploadChunkAction&jsonp=1',
    completeUrl: 'json.php?mod=thirdWH&act=completeUploadAction&jsonp=1',
    onProgress: function (percent) {
      // console.log('上传进度:', percent + '%');
      // 更新加载层的提示文字
      $('#uploadProgressText').text('文件上传中 (' + percent + '%)');
    },
    onSuccess: function (result) {
      console.log('上传成功:', result);
      // 关闭loading
      layer.close(loadIndex2);
      // 上传成功后提交表单数据
      submitFormData(result.data.path);
    },
    onError: function (error) {
      console.error('上传失败:', error);
      // 关闭loading
      layer.close(loadIndex2);
      layer.msg('上传失败: ' + error.message, { icon: 2 });
      resetUploadStatus();
    }
  });
  
  // 异步上传文件
  uploader.upload(selectedFile).catch(function (error) {
    console.error('上传过程发生错误:', error);
    layer.close(loadIndex2);
  });
  ```

- 后端实现分片上传接收接口

  ```php
  public function act_initUploadAction()
  {
      $fileName = empty($_POST['fileName']) ? '' : $_POST['fileName'];
      $fileSize = empty($_POST['fileSize']) ? 0 : $_POST['fileSize'];
  
      $result = service('chunkUpload')->initUpload([
          'fileName' => $fileName,
          'fileSize' => $fileSize
      ]);
      if($result === false){
          self::$errCode = 201;
          self::$errMsg = service('chunkUpload')->getError();
          return false;
      }
  
      self::$errCode = 200;
      self::$errMsg = "初始化成功";
      return $result;
  }
  
  /**
   * 上传分片
   */
  public function act_uploadChunkAction()
  {
      $uploadId = empty($_POST['uploadId']) ? '' : $_POST['uploadId'];
      $chunkIndex = empty($_POST['chunkIndex']) ? 0 : $_POST['chunkIndex'];
      $file = empty($_FILES['chunk']) ? null : $_FILES['chunk'];
  
      if (!$uploadId || !$file) {
          self::$errCode = 201;
          self::$errMsg = "参数错误";
          return false;
      }
  
  
      $result = service('chunkUpload')->uploadChunk($uploadId, $chunkIndex, $file);
      if($result === false){
          self::$errCode = 201;
          self::$errMsg = service('chunkUpload')->getError();
          return false;
      }
  
      self::$errCode = 200;
      self::$errMsg = "成功";
      return $result;
  }
  
  /**
   * 完成上传
   */
  public function act_completeUploadAction()
  {
      $uploadId = empty($_POST['uploadId']) ? '' : $_POST['uploadId'];
      $fileName = empty($_POST['fileName']) ? '' : $_POST['fileName'];
  
      if (!$uploadId || !$fileName) {
          self::$errCode = 201;
          self::$errMsg = "参数错误";
          return false;
      }
  
      // 生成目标路径
      $targetDir = WEB_PATH . 'html/upload/bol/';
      $targetPath = $targetDir . date('Ymd') . '/' . uniqid() . '_' . $fileName;
      $result = service('chunkUpload')->completeUpload($uploadId, $fileName, $targetPath);
      if($result === false){
          self::$errCode = 201;
          self::$errMsg = service('chunkUpload')->getError();
          return false;
      }
  
      self::$errCode = 200;
      self::$errMsg = "上传成功";
      return $result;
  }
  ```

### 前端完整代码

- html表单代码

  ```html
  <div style="display: none; margin: 10px;" id="upload_bol_window">
      <form class="layui-form" id="uploadForm">
  
        <input type="hidden" id="upload_bol_id" name="upload_bol_id">
        <!-- 文件上传区域 -->
        <div class="layui-form-item">
          
          <label class="layui-form-label">PDF文件：</label>
          <div class="layui-input-inline">
            <div class="layui-upload-drag" id="fileUploadArea">
              <i class="layui-icon layui-icon-upload"></i>
              <p>点击上传或将文件拖拽到此处</p>
              <div class="layui-hide" id="uploadPreview">
                <hr>
                <p id="selectedFileName"></p>
              </div>
            </div>
          </div>
        </div>
  
        <!-- 运费和币种 -->
        <div class="layui-form-item">
          <label class="layui-form-label">运费</label>
          <div class="layui-input-inline">
            <input type="text" name="fees" id="bol_fees" required lay-verify="required|number" placeholder="请输入运费"
              autocomplete="off" class="layui-input">
            <div class="layui-form-mid layui-word-aux">币种：USD</div>
          </div>
          
        </div>
  
        <!-- 备注 -->
        <div class="layui-form-item layui-form-text layui-row">
          <label class="layui-form-label">备注</label>
          <div class="layui-input-inline">
            <textarea name="bol_remark" placeholder="请输入备注内容" id="bol_remark" class="layui-textarea"
              style="width: 100%; height: 100px;"></textarea>
            <div class="layui-form-mid layui-word-aux">备注LTL快递</div>
          </div>
          
          
        </div>
  
        <!-- 操作按钮 -->
        <div class="layui-form-item">
          <div class="layui-input-block">
            <button type="button" class="layui-btn" id="submitUpload"><i class="layui-icon layui-icon-upload-circle"></i>
              确定上传</button>
            <button type="reset" class="layui-btn layui-btn-primary" id="cancelUpload">取消</button>
            <button type="button" class="layui-btn layui-btn-warm layui-hide" id="pauseUpload"><i
                class="layui-icon layui-icon-pause"></i> 暂停</button>
            <button type="button" class="layui-btn layui-btn-normal layui-hide" id="resumeUpload"><i
                class="layui-icon layui-icon-play"></i> 继续</button>
          </div>
        </div>
      </form>
    </div><div style="display: none; margin: 10px;" id="upload_bol_window">
      <form class="layui-form" id="uploadForm">
  
        <input type="hidden" id="upload_bol_id" name="upload_bol_id">
        <!-- 文件上传区域 -->
        <div class="layui-form-item">
          
          <label class="layui-form-label">PDF文件：</label>
          <div class="layui-input-inline">
            <div class="layui-upload-drag" id="fileUploadArea">
              <i class="layui-icon layui-icon-upload"></i>
              <p>点击上传或将文件拖拽到此处</p>
              <div class="layui-hide" id="uploadPreview">
                <hr>
                <p id="selectedFileName"></p>
              </div>
            </div>
          </div>
        </div>
  
        <!-- 运费和币种 -->
        <div class="layui-form-item">
          <label class="layui-form-label">运费</label>
          <div class="layui-input-inline">
            <input type="text" name="fees" id="bol_fees" required lay-verify="required|number" placeholder="请输入运费"
              autocomplete="off" class="layui-input">
            <div class="layui-form-mid layui-word-aux">币种：USD</div>
          </div>
          
        </div>
  
        <!-- 备注 -->
        <div class="layui-form-item layui-form-text layui-row">
          <label class="layui-form-label">备注</label>
          <div class="layui-input-inline">
            <textarea name="bol_remark" placeholder="请输入备注内容" id="bol_remark" class="layui-textarea"
              style="width: 100%; height: 100px;"></textarea>
            <div class="layui-form-mid layui-word-aux">备注LTL快递</div>
          </div>
          
          
        </div>
  
        <!-- 操作按钮 -->
        <div class="layui-form-item">
          <div class="layui-input-block">
            <button type="button" class="layui-btn" id="submitUpload"><i class="layui-icon layui-icon-upload-circle"></i>
              确定上传</button>
            <button type="reset" class="layui-btn layui-btn-primary" id="cancelUpload">取消</button>
            <button type="button" class="layui-btn layui-btn-warm layui-hide" id="pauseUpload"><i
                class="layui-icon layui-icon-pause"></i> 暂停</button>
            <button type="button" class="layui-btn layui-btn-normal layui-hide" id="resumeUpload"><i
                class="layui-icon layui-icon-play"></i> 继续</button>
          </div>
        </div>
      </form>
    </div>
  <!--引入工具类-->
  <script type="text/javascript" src="js/common/chunkUpload.js?v=20250327"></script>
  ```

- js代码

  ```js
  layui.use(function () {
    var layer = layui.layer
    var $ = layui.jquery
    var form = layui.form
    var formIndex = null
    var table = layui.table
    var laydate = layui.laydate;
  	//上传平台自提文件优化
    let selectedFile = null;
    let uploader = null;
    let isPaused = false;
    $('#uploadBolBtn').click(function(){
      var itemJq = $("input[name='items[]']:checked");
      if(itemJq.length != 1){
        layer.msg('请勾选一个需要上传的备货单', {
          icon: 2,
          time: 2000
        });
        return false;         
      }
      
      itemJq.each(function (idx, ele) {
        var id = $(ele).data('id');
        var status = $(ele).data('status');
        var remark = $(ele).data('remark');
        var fees = $(ele).data('fees');
        var status_arr = ["ORDER_HAD_BOXED", "ORDERS_WAIT_TRANSFER", "ORDERS_FINISHED", "ORDERS_SHIPPING","ORDERS_TRANSFERED","ORDER_INIT"];
        var re_sta = status_arr.indexOf(status)
  
        if (re_sta <= -1) {
          showError('请选择装箱完成,待中转,已中转,发货中,完结订单状态的备货单!');
          flag = 1;
        }
        $('#upload_bol_id').val(id);
        $('#bol_remark').val(remark);
        $('#bol_fees').val(fees);
      })
      
      // 初始化上传组件
      initFileUpload();
      
      layer.open({
        type: 1,
        title: '上传平台自提文件',
        area: ['600px', '575px'],
        content: $('#upload_bol_window'),
        success: function (layero, index, that) {
          // 重置上传状态
          resetUploadStatus();
          $('#uploadPreview').addClass('layui-hide');
          $('#selectedFileName').text('');
          $('input[type="file"]').val('');
          $('#fileUploadArea .layui-icon-upload').show();
          $('#fileUploadArea p:first').show();
          selectedFile = null;
          uploader = null;
          isPaused = false;
        },
        cancel: function (index, layero, that) {
          $("#update_skuqty_form")[0].reset();
          form.val("update_skuqty_form", {});
          form.render(null, receive_address_form);
        }
      })
    })
  
    // 提交上传按钮
    $('#submitUpload').on('click', function () {
      if (!selectedFile) {
        layer.msg('请先选择文件', { icon: 2 });
        return;
      }
  
      // 表单验证
      let fees = $('#bol_fees').val();
      if (!fees) {
        layer.msg('请输入运费', { icon: 2 });
        return;
      }
  
      // 隐藏提交按钮，显示暂停按钮
      $('#pauseUpload').removeClass('layui-hide');
      $('#submitUpload').addClass('layui-hide');
  
      // 创建自定义加载层，包含进度显示
      var loadingHtml = '<div style="padding: 20px; line-height: 22px; text-align: center;">' +
        '<i class="layui-icon layui-icon-loading layui-anim layui-anim-rotate layui-anim-loop" style="font-size: 30px;"></i>' +
        '<div id="uploadProgressText" style="margin-top: 15px;">文件上传中 (0%)</div>' +
        '</div>';
  
      var loadIndex2 = layer.open({
        type: 1,
        title: false,
        closeBtn: 0,
        shade: [0.3, '#fff'],
        area: ['300px', '100px'],
        content: loadingHtml
      });
      
      // 创建上传实例
      uploader = new ChunkUploader({
        initUrl: 'json.php?mod=thirdWH&act=initUploadAction&jsonp=1',
        uploadUrl: 'json.php?mod=thirdWH&act=uploadChunkAction&jsonp=1',
        completeUrl: 'json.php?mod=thirdWH&act=completeUploadAction&jsonp=1',
        onProgress: function (percent) {
          // console.log('上传进度:', percent + '%');
          // 更新加载层的提示文字
          $('#uploadProgressText').text('文件上传中 (' + percent + '%)');
        },
        onSuccess: function (result) {
          console.log('上传成功:', result);
          // 关闭loading
          layer.close(loadIndex2);
          // 上传成功后提交表单数据
          submitFormData(result.data.path);
        },
        onError: function (error) {
          console.error('上传失败:', error);
          // 关闭loading
          layer.close(loadIndex2);
          layer.msg('上传失败: ' + error.message, { icon: 2 });
          resetUploadStatus();
        }
      });
  
      // 异步上传文件
      uploader.upload(selectedFile).catch(function (error) {
        console.error('上传过程发生错误:', error);
        layer.close(loadIndex2);
      });
    });
  
    // 暂停上传
    $('#pauseUpload').on('click', function () {
      if (uploader && uploader.pause) {
        uploader.pause();
        isPaused = true;
        $('#pauseUpload').addClass('layui-hide');
        $('#resumeUpload').removeClass('layui-hide');
      }
    });
  
    // 继续上传
    $('#resumeUpload').on('click', function () {
      if (uploader && uploader.resume) {
        // 显示继续上传的加载层
        var resumeLoadingIndex = layer.msg('继续上传中...', {
          icon: 16,
          shade: [0.3, '#fff'],
          time: 0
        });
  
        uploader.resume();
        isPaused = false;
        $('#resumeUpload').addClass('layui-hide');
        $('#pauseUpload').removeClass('layui-hide');
  
        // 2秒后关闭提示
        setTimeout(function () {
          layer.close(resumeLoadingIndex);
        }, 2000);
      }
    });
  
    // 初始化文件上传
    function initFileUpload() {
      // 拖拽上传区域
      layui.upload.render({
        elem: '#fileUploadArea',
        url: '', // 不设置URL，我们将使用自定义的上传方法
        auto: false, // 不自动上传
        accept: 'file',
        exts: 'pdf|zip|rar', // 限制文件类型
        choose: function (obj) {
          console.log('show loading');
          var loadIndex1 = layer.load(1, {
            shade: [0.3, '#fff']
          });
          obj.preview(function (index, file, result) {
            selectedFile = file;
            $('#selectedFileName').text('已选择: ' + file.name);
            $('#uploadPreview').removeClass('layui-hide');
  
            $('#fileUploadArea .layui-icon-upload').hide();
            $('#fileUploadArea p:first').hide();
  
            // 如果有loading，关闭它
            if (loadIndex1) {
              layer.close(loadIndex1);
            }
          });
        },
      });
    }
    // 重置上传状态
    function resetUploadStatus() {
      $('#pauseUpload').addClass('layui-hide');
      $('#resumeUpload').addClass('layui-hide');
      $('#submitUpload').removeClass('layui-hide');
    }
  
    // 提交表单数据
    function submitFormData(filePath) {
      let formData = {
        id: $('#upload_bol_id').val(),
        fees: $('#bol_fees').val(),
        remark: $('#bol_remark').val(),
        file_path: filePath
      };
  
      $.ajax({
        url: 'json.php?mod=thirdWH&act=saveBolInfo&jsonp=1',
        type: 'POST',
        data: formData,
        dataType: 'json',
        success: function (res) {
          console.log(res);
          if (res.errCode == 200) {
            layer.msg('上传成功', { icon: 1 });
            layer.closeAll('page');
  
          } else {
            layer.msg(res.errMsg || '保存失败', { icon: 2 });
          }
        },
        error: function () {
          layer.msg('网络错误，请重试', { icon: 2 });
        },
        complete: function () {
          resetUploadStatus();
        }
      });
    }
  })
  ```

  
## Prerequisites
- Cloud Storage is activated.
- The AccessKey and SecretKey are created
- System: above PHP5.4


## Install
1, Manage project dependencies through composer
```
"require": {
    "wangsucs/wcs-sdk-php": "^2.0.0"
}

```


2, You can also download [PHP SDK](https://wcsd.chinanetcenter.com/sdk/cnc-php-sdk-wcs.zip), and then import manually.
```
require_once __DIR__ . '/vendor/autoload.php';


```

## Initialization
When accessing the cloud storage, users need to use a valid pair of AccessKey and SecretKey for signature authentication, and add information such as "upload domain" and "manage domain". The configuration information only needs to be initialized once throughout the application, as follows:
```
/*src/Wcs/Config.php*/
//url settings
$WCS_PUT_URL    = 'your uploadDomain';
$WCS_GET_URL    = 'your downloadDomain';
$WCS_MGR_URL	= 'your mgrDomain';

//access key and secret key settings
$WCS_ACCESS_KEY	= 'your access key';
$WCS_SECRET_KEY	= 'your secrete key';

//deadline of token, default is 1 hour (3600s)
const  WCS_TOKEN_DEADLINE = 3600;

//Upload file settings
const WCS_OVERWRITE = 0; //default is not overwrite

//time limit exceeded
const WCS_TIMEOUT = 20;

//chunk upload para settings
const WCS_BLOCK_SIZE = 4 * 1024 * 1024; //default block size is 4M
const WCS_CHUNK_SIZE = 256 * 1024; //Default chunk size is 256K
const WCS_RECORD_URL = './'; //current file directory as default
const WCS_COUNT_FOR_RETRY = 3;  //Timeout retry count

```

## Function Explanation
### Normal Upload
Noraml upload takes the sheet form upload method to directly upload files to cloud storage. It is recommended to use this method when the file is less than 20M.
- If the ==callbackBody== parameter is specified, WCS makes an HTTP request to the server address specified by the ==callbackUrl==. 
The server completes processing according to this content, and customizes the response content in the HTTP response. After receiving the response, WCS will send the feedback data to the client. If the ==callbackBody== parameter is not specified, WCS returns an empty string to the client.
- If you want setup a pre-processing in a successful upload, it is specified with ==persistentOps== parameter. ==persistentNotifyUrl== specifies the method to handle the successful upload, and if you specify ==returnBody== and ==returnUrl==, WCS will go to the address specified by ==returnUrl== and pass the parameters specified by ==returnBody== after the successful upload. 

#### Example
```
//bucketName 
//fileKey   
//localFile 
//returnBody    Customize the return content (optional)
//userParam customized variable name    <x:VariableName>    (optional)
//userVars  customized variable value   <x:VariableValue>   (optional)
//mimeType  customized upload type (optional) 

require '../vendor/autoload.php';
use Wcs\Upload\Uploader;
use Wcs\Http\PutPolicy;

$pp = new PutPolicy();
if ($fileKey == null || $fileKey === '') {
    $pp->scope = $bucketName;
} else {
    $pp->scope = $bucketName . ':' . $fileKey;
}

// Notification upload
$pp->returnBody = '';
$pp->returnUrl = '';

// Callback upload
$pp->callbackBody = '';
$pp->callbackUrl = '';

// Pre-process
$pp->persistentOps = '<cmd>';

// validity period of token
$pp->deadline = '';//unit is ms
$token = $pp->get_token();

$client = new Uploader($token, $userParam, $userVars, $mimeType);
$resp = $client->upload_return($localFile);
print_r($resp);

```
####  Command Line Test
```
$ php file_upload_return.php [-h | --help] -b <bucketName> -f <fileKey> -l <localFile> [-r <returnBody>] [-u <userParam>] [-v <userVars>] [-m <mimeType>]

$ php file_upload_callback.php [-h | --help] -b <bucketName> -f <fileKey> -l <localFile> -c <callbackUrl> [-r <returnBody>] [-u <userParam>] [-v <userVars>] [-m <mimeType>]

$ php file_upload_notify.php [-h | --help] -b <bucketName> -f <fileKey> -l <localFile> -n <notifyUrl> -c <cmd> [-u <userParam>] [-v <userVars>] [-m <mimeType>]

```

### Multipart Upload
The general process of multipart upload is as follows:

1, mkblk (mknlk must be operated before each block is uploaded, and the server returns the first CTX)
2, bput (bput after mkblk, upload block with the previous CTX and return the current CTX)
3, mkfile (when the file is uploaded, do mkfile with the last CTX information for each block)

**Note:**
- In case of request timeout,  multipart upload will be retransmitted automatically. In other cases (e.g. the status code is not 28), report an error and exit, and the error information will be saved in the current directory. Save as Filename. Log (hidden file).
- After the upload is interrupted, the upload information is saved in a hidden file “Filename.rcd”, each item record the information of block upload. Breakpoint upload will analyze the status of the current upload from the last recorded information and make subsequent uploads. After that, the record file will be deleted.
- For breakpoint upload, you only need to re-execute the multipart upload operation.
- Multipart upload only does concurrency inside the block, and it is initialization concurrency instead of multithreading. Considering that PHP doesn't support multithreading very well, the conversion mechanism is implemented using guzzleHTTP.
- The default size of the block is 4M and the size of the chunk is 256K, which is for more stable upload. If user thinks the upload speed is too slow and wants to improve it, he only need to adjust the size of the block or chunk, which can improve the upload speed (relatively, the upload stability may be reduced).
- Due to the time-out retransmission strategy (at least 3 retry times) to ensure the reliability of the transmission, if the customer wants to accelerate the upload speed, the size of the chunk can be converted to the size of the block to ensure the maximum number of concurrency and improve the upload speed.
- Multipart upload progress information is in "filename.rcd" , save as JSON. It will write a JSON record after uploading a chunk, and the progress information is stored in the $JSON [' Info '] [' Progress '] partition, and the customer can process this infomation as needed.
- If the upload is successful, filename.rcd and filename.log will be deleted. 

#### Vaiable Explanation
```
//Basic Information
private $blockSize;
private $chunkSize;
private $countForRetry;
private $timeoutForRetry;

//User customized information
private $userParam;
private $encodedUserVars;
private $mimeType;

//uuid random number use php uniqid()
private $uuid;

//breakpoint upload record file
private $recordFile;

//breakpoint uoload information
private $localFile;
private $blockNumOfUploaded;    //uploaded block number
private $chunkNumOfUploaded;    //uploaded chunk number in current block
private $ctxListForMkfile;  //the last ctx of each chunk in mkfile oprtaion
private $sizeOfFile;    //file size
private $sizeOfUploaded;    //size of file upload
private $latestChunkCtx;    //the latest ctx
private $time;  //generation time of token, it is used to verify if the token is valid

```

#### Example
```
//bucketName buck name
//fileKey   customized file name
//localFile  name of file uploaded

require '../vendor/autoload.php';
use Wcs\Upload\ResumeUploader;
use Wcs\Http\PutPolicy;

$pp = new PutPolicy();
if ($fileKey == null || $fileKey === '') {
    $pp->scope = $bucketName;
} else {
    $pp->scope = $bucketName . ':' . $fileKey;
}
$pp->deadline = '1483027200000';
$pp->persistentOps = $cmd;
$pp->persistentNotifyUrl = $notifyUrl;
$pp->returnBody = $returnBody;
$token = $pp->get_token();

$client = new ResumeUploader($token, $userParam, $encodeUserVars, $mimeType);
$client->upload($localFile);

```

### Resource Management
Provide the basic operation to files.

#### Delete file
##### Example
```
require '../vendor/autoload.php';
use Wcs\SrcManage\FileManager;
use Wcs\MgrAuth;
use Wcs\Config;

$ak = Config::WCS_ACCESS_KEY
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new FileManager($auth);
print_r($client->delete($bucketName, $fileKey));


```

##### Command Line Test
```
$ php file_delete.php [-h | --help] -b <bucketName> -f <fileKey>

```

#### Get file info

##### Example
```
require '../vendor/autoload.php';
use Wcs\SrcManage\FileManager;
use Wcs\MgrAuth;
use Wcs\Config;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new FileManager($auth);
print_r($client->stat($bucketName, $fileKey));

```

##### Command Line Test
```
$ php file_stat.php [-h | --help] -b <bucketName> -f <fileKey>


```

#### Dual Resource

##### Example
```
require '../vendor/autoload.php';
use Wcs\SrcManage\FileManager;
use Wcs\MgrAuth;
use Wcs\Config;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new FileManager($auth);
print_r($client->bucketList($bucketName, $limit, $prefix, $mode, $marker));

```

##### Command Line Test
```
$ php file_download.php [-h | --help] -b <bucketName> [-l <limit>] [-p <prefix>] [-m <mode>] [--ma <marker>]

```

#### Update Supplemental Resource
##### Example
```
require '../vendor/autoload.php';
use Wcs\SrcManage\FileManager;
use Wcs\MgrAuth;
use Wcs\Config;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

//fileKeys = "<fileKey1>|<fileKey2>|<fileKey3>";
$client = new FileManager($auth);
print_r($client->updateMirrorSrc($bucketName, $fileKeys));

```

##### Command Line Test
```
$ php file_stat.php [-h | --help] -b <bucket> -f [<fileKey1>|<fileKey2>|<fileKey3>...]

```

#### Move Resource
##### Example
```
require '../vendor/autoload.php';
use Wcs\SrcManage\FileManager;
use Wcs\MgrAuth;
use Wcs\Config;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new FileManager($auth);
print_r($client->move($bucketSrc, $keySrc, $bucketDst, $keyDst));

```

##### Command Line Test
```
$ php file_move.php [-h | --help] --bs <bucketSrc> --ks <keyStr> --bd <bucketDst> --kd <keyDst>

```

#### Copy Resource
##### Example
```
require '../vendor/autoload.php';
use Wcs\SrcManage\FileManager;
use Wcs\MgrAuth;
use Wcs\Config;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new FileManager($auth);
print_r($client->copy($bucketSrc, $keySrc, $bucketDst, $keyDst));

```

##### Command Line Test
```
$ php file_copy.php [-h | --help] --bs <bucketSrc> --ks <keyStr> --bd <bucketDst> --kd <keyDst>

```

#### Get metadata of audio/video file
##### Example
```
require '../vendor/autoload.php';
use Wcs\SrcManage\FileManager;
use Wcs\MgrAuth;
use Wcs\Config;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new FileManager($auth);
print_r($client->avInfo($key));

```
##### Command Line Test
```
php avinfo.php [-h | --help] -k <key>

```

#### Get simple metadata of audio/video file

##### Example
```
require '../vendor/autoload.php';
use Wcs\SrcManage\FileManager;
use Wcs\MgrAuth;
use Wcs\Config;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new FileManager($auth);
print_r($client->avInfo2($key));

```
##### Command Line Test
```
$ php avinfo2.php [-h | --help] -k <key>

```

#### Set expiry for files
##### Example
```
require '../../vendor/autoload.php';
use Wcs\SrcManage\FileManager;
use Wcs\MgrAuth;
use Wcs\Config;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new FileManager($auth);
print_r($client->setDeadline($bucketName, $fileKey, $deadline));

```
##### Command Line Test
```
$ php file_setDeadLine.php [-h | --help] -b <bucketName> -f <fileKey> -d <deadline>

```

#### Audio/Video Processing
##### fops opertion
```
require '../../vendor/autoload.php';
use Wcs\PersistentFops\Fops;
use Wcs\Config;
use Wcs\MgrAuth;

//$the format of fops

$bucket = '<input key>';
$key = '<input key>';

//parameter setting
$notifyURL = '';
$force = 0;
$separate = 0;

$fops = '';

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new Fops($auth, $bucket);
print_r($client->exec($fops, $key, $notifyURL, $force, $separate));

```

##### Query fops
```
require '../../vendor/autoload.php';
use Wcs\PersistentFops\Fops;
print_r(Fops::status($persisetntId));

```

### Advanced Resource Management

#### Get resource
```
require '../../vendor/autoload.php';
use Wcs\Fmgr\Fmgr;
use Wcs\Config;
use Wcs\MgrAuth;
use Wcs\Utils;

//optional parameters
$notifyURL = '';
$force = 0;
$separate  = 0;

//fops parameters
$fetchURL = Utils::url_safe_base64_encode('https://www.baidu.com/img/bd_logo1.png');
$bucket = Utils::url_safe_base64_encode('<input key>');
$key = Utils::url_safe_base64_encode('<input key>');
$prefix = Utils::url_safe_base64_encode('<input key>');

$fops = 'fops=fetchURL/'.$fetchURL.'/bucket/'.$bucket.'/key/'.$key.'/prefix/'.$prefix.'&notifyURL='.Utils::url_safe_base64_encode($notifyURL).'&force='.$force.'&separate='.$separate;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new Fmgr($auth, $notifyURL, $force, $separate);
print_r($client->fetch($fops));

```

#### Copy Resource
```

require '../../vendor/autoload.php';
use Wcs\Fmgr\Fmgr;
use Wcs\Config;
use Wcs\MgrAuth;
use Wcs\Utils;

//optional parameters
$notifyURL = '';
$force = 0;
$separate  = 0;

//fops parameters
$resource = Utils::url_safe_base64_encode('<input key>');
$bucket = Utils::url_safe_base64_encode('<input key>');
$key = Utils::url_safe_base64_encode('<input key>');
$prefix = Utils::url_safe_base64_encode('<input key>');

$fops = 'fops=resource/'.$resource.'/bucket/'.$bucket.'/key/'.$key.'/prefix/'.$prefix.'&notifyURL='.Utils::url_safe_base64_encode($notifyURL).'&force='.$force.'&separate='.$separate;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new Fmgr($auth, $notifyURL, $force, $separate);
print_r($client->copy($fops));

```

#### Move Resource
```

require '../../vendor/autoload.php';
use Wcs\Fmgr\Fmgr;
use Wcs\Config;
use Wcs\MgrAuth;
use Wcs\Utils;

//optional parameters
$notifyURL = '';
$force = 0;
$separate  = 0;

//fops parameters
$resource = Utils::url_safe_base64_encode('<input key>');
$bucket = Utils::url_safe_base64_encode('<input key>');
$key = Utils::url_safe_base64_encode('<input key>');
$prefix = Utils::url_safe_base64_encode('<input key>');

$fops = 'fops=resource/'.$resource.'/bucket/'.$bucket.'/key/'.$key.'/prefix/'.$prefix.'&notifyURL='.Utils::url_safe_base64_encode($notifyURL).'&force='.$force.'&separate='.$separate;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new Fmgr($auth, $notifyURL, $force, $separate);
print_r($client->move($fops));

```

#### Delete resource
```

require '../../vendor/autoload.php';
use Wcs\Fmgr\Fmgr;
use Wcs\Config;
use Wcs\MgrAuth;
use Wcs\Utils;

//optional parameters
$notifyURL = '';
$force = 0;
$separate  = 0;

//fops parameters
$bucket = Utils::url_safe_base64_encode('<input key>');
$key = Utils::url_safe_base64_encode('<input key>');

$fops = 'fops=bucket/'.$bucket.'/key/'.$key.'&notifyURL='.Utils::url_safe_base64_encode($notifyURL).'&force='.$force.'&separate='.$separate;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new Fmgr($auth, $notifyURL, $force, $separate);
print_r($client->delete($fops));

```

#### Delete resource by pre-set info
```

require '../../vendor/autoload.php';
use Wcs\Fmgr\Fmgr;
use Wcs\Config;
use Wcs\MgrAuth;
use Wcs\Utils;

//optional parameters
$notifyURL = '';
$force = 0;
$separate  = 0;

//fops parameters
$bucket = Utils::url_safe_base64_encode('<input key>');
$prefix = Utils::url_safe_base64_encode('<input key>');
$output = Utils::url_safe_base64_encode('<input key>');

$fops = 'fops=bucket/'.$bucket.'/prefix/'.$prefix.'&notifyURL='.Utils::url_safe_base64_encode($notifyURL).'&force='.$force.'&separate='.$separate;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new Fmgr($auth, $notifyURL, $force, $separate);
print_r($client->deletePrefix($fops));

```

#### fmgr Task Query
```

require '../../vendor/autoload.php';
use Wcs\Fmgr\Fmgr;
use Wcs\Config;
use Wcs\MgrAuth;

//optional parameter
$notifyURL = '';
$force = 0;
$separate  = 0;

$ak = Config::WCS_ACCESS_KEY;
$sk = Config::WCS_SECRET_KEY;
$auth = new MgrAuth($ak, $sk);

$client = new Fmgr($auth, $notifyURL, $force, $separate);
print_r($client->status("<input persistentId>"));

```

```

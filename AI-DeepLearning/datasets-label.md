## labelme vs others
- [semantic-segmentation-editor](https://github.com/Hitachi-Automotive-And-Industry-Lab/semantic-segmentation-editor)
- [labelme](https://github.com/wkentaro/labelme)

### Labelme

[labelme](https://github.com/wkentaro/labelme): qt based passed 

### Labelme-web 
[labelme-web](http://labelme.csail.mit.edu/Release3.0/)

web-based（参考资料简单、少）

github code: https://github.com/CSAILVision/LabelMeToolbox

https://github.com/CSAILVision/LabelMeAnnotationTool

usage:https://blog.roboflow.com/labelme/

step1: 注册账号并登陆 http://labelme2.csail.mit.edu/Release3.0/browserTools/php/browse_collections.php?username=hayley&folder=

step2: 上传图片到labelme（缺点：一次最多上传20张，只支持jpg，图片大小有限制5M以下） 

step3: 开始标注（生成的标注信息是xml格式）

step4: 从labelme下载数据（将图片和标注压缩成zip包）

step5: 将上一步下载的解压，上传到lroboflow（需要注册）中。可以将标注转换成别的模型识别的格式。或者对数据进行预处理等，也可以用里面的模型对数据进行。
页面风格和tusai的风格不是很搭


### semantic-segmentation-editor
[semantic-segmentation-editor](https://github.com/Hitachi-Automotive-And-Industry-Lab/semantic-segmentation-editor)

web-based
meteor app, react,paper.js, three.js
最新版本1.5.3 提供了docker image 

使用步骤：

step1: 安装并启动.默认端口为80
```markdown
wget https://raw.githubusercontent.com/Hitachi-Automotive-And-Industry-Lab/semantic-segmentation-editor/master/sse-docker-stack.yml
SSE_IMAGES=/Users/hayley/Desktop/semantic-segmentation-editor/images 
docker-compose -f sse-docker-stack.yml up

met error
ERROR: for semantic-segmentation-editor_app_1  Cannot create container for service app: create .: volume name is too short, names should be at least two alphanumeric characters
fix:创建data目录，并且修改volumes值
    volumes:
      - "${PWD}/images/:/root/sse-images:rw"

```
step2: 打开浏览器http://localhost/browse/0/20/
step3: 标注,生成json标注信息

### VIA
 - http://www.robots.ox.ac.uk/~vgg/software/via/via.html
 - http://www.robots.ox.ac.uk/~vgg/software/via/
2017-11-02

## openstack 国际化和本地化介绍和配置

Openstack所有子项目的本地化工作主要由oslo_i18n项目实现，使用oslo_i18n的规范为：

* 在项目根下创建i18n.py模块。
* 在模块中声明domain，如nova、cinder等。
* 根据domain  创建TranslatorFactory实例。
* 使用TranslatorFactory的primary作为默认翻译转化函数。
* 如果不使用默认的翻译转化函数，你还可以使用oslo_i18n.translate(msg, my_locale)方法覆盖。

比如我们有一个新的项目my_app，需要使用oslo_i18n本地化和国际化，创建i18n.py模块为:
```
# myapp/_i18n.py

import oslo_i18n

DOMAIN = "myapp"

_translators = oslo_i18n.TranslatorFactory(domain=DOMAIN)

# The primary translation function using the well-known name "_"
_ = _translators.primary

# The contextual translation function using the name "_C"
# requires oslo.i18n >=2.1.0
_C = _translators.contextual_form

# The plural translation function using the name "_P"
# requires oslo.i18n >=2.1.0
_P = _translators.plural_form

# Translators for log levels.
#
# The abbreviated names are meant to reflect the usual use of a short
# name like '_'. The "L" is for "log" and the other letter comes from
# the level.
_LI = _translators.log_info
_LW = _translators.log_warning
_LE = _translators.log_error
_LC = _translators.log_critical


def get_available_languages():
    return oslo_i18n.get_available_languages(DOMAIN)
```
以上的_开头的都是变量的简写，方便后续引用。
剩下的代码只需要引用对应变量即可:
```
from myapp._i18n import _, _LW, _LE

# ...

variable = "openstack"
LOG.warning(_LW('warning message: %s'), variable)

# ...

try:

    # ...

except AnException1:

    # Log only
    LOG.exception(_LE('exception message'))

except AnException2:

    # Raise only
    raise RuntimeError(_('exception message'))

else:

    # Log and Raise
    msg = _('Unexpected error message')
    LOG.exception(msg)
    raise RuntimeError(msg)
```
获取支持的所有语言列表:
```
import myapp._i18n

languages = myapp._i18n.get_available_languages()
```
本地化配置使用的是系统变量：

* LANGUAGE
* LC_ALL
* LC_MESSAGES

以上变量从上到下的优先级读取。

如果仅仅需要设置某个组件的本地化，比如nova-compute，修改openstack-nova-compute.service文件，在[Service]下增加以下环境变量:
```
Environment=LANGUAGE=zh_CN.utf-8
Environment=LC_ALL=zh_CN.utf-8
```
本地化使用mo文件通常在/usr/share/locale目录下:
```
find /usr/share/locale/ -name "*.mo"
```
实现源码通常在项目的locale目录,如nova源码为nova/locale。


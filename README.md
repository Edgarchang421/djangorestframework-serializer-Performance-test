#  djangorestframework-serializer-Performance-test  
本文章參考自 : https://hakibenita.com/django-rest-framework-slow  
主要測試各種不同djangorestframework提供的serializer的效能  
  
#  前置作業  
前置作業參考 : https://www.django-rest-framework.org/tutorial/quickstart/  
主要使用版本 : Python 3.7.3、django 2.2.7、djangorestframework 3.10.3  

#  測試  
這個測試主要為測試不同的serializer寫法的效能  
使用Django中django.contrib.auth.models的User來進行各種serializer的效能測試  
  
為了單獨測試serializer的效能，所以只會單獨使用一個instance做serialize，避免database的外在因素影響，也不會完整的使用到API的功能  
  
##  測試一  
雖然不用建立django app也可以測試，但我還是建立了main app後，在main中建立serializers.py檔案，將serializer建立在裡面  
  
測試一將不使用任何rest_framework提供的serializer  
單純將User instance從django.contrib.auth.models.User class轉換成dictionary的資料類型  
然後return  
```
from typing import Dict, Any

from django.contrib.auth.models import User


def serialize_user(user: User) -> Dict[str, Any]:
    return {
        'id': user.id,
        'last_login': user.last_login.isoformat() if user.last_login is not None else None,
        'is_superuser': user.is_superuser,
        'username': user.username,
        'first_name': user.first_name,
        'last_name': user.last_name,
        'email': user.email,
        'is_staff': user.is_staff,
        'is_active': user.is_active,
        'date_joined': user.date_joined.isoformat(),
    }
```
使用cProfile來進行測試  
關於cProfile : https://docs.python.org/3/library/profile.html  
```
>>> from django.contrib.auth.models import User
>>> import cProfile
>>> from main.serializers import serialize_user
>>> u = User.objects.create_user(username="username")
>>> cProfile.runctx('for i in range(5000):serialize_user(u)',globals(),locals(),sort="tottime")
         15003 function calls in 0.025 seconds

   Ordered by: internal time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     5000    0.012    0.000    0.013    0.000 {method 'isoformat' of 'datetime.datetime' objects}
        1    0.007    0.007    0.025    0.025 <string>:1(<module>)
     5000    0.005    0.000    0.018    0.000 serializers.py:7(serialize_user)
     5000    0.001    0.000    0.001    0.000 __init__.py:223(utcoffset)
        1    0.000    0.000    0.025    0.025 {built-in method builtins.exec}
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}

```
僅僅使用了0.025秒就完成了5000次的serialize  
##  測試二  
改為使用rest_framework提供的ModelSerializer，ModelSerializer提供非常多實用的功能  
* 基於要serialize的models，自動生成Field  
* 自動生成validator，例如 : UniqueValidator  
* 提供 .create() 和 .update() 這兩個API常用的method  
  
關於ModeSerializer : https://www.django-rest-framework.org/api-guide/serializers/#modelserializer  
關於Validator : https://www.django-rest-framework.org/api-guide/validators/  
```
from rest_framework import serializers

class UserModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id','last_login','is_superuser','username','first_name','last_name','email','is_staff','is_active','date_joined']
```
使用cProfile來進行測試  
```
>>> from main.serializers import UserModelSerializer
>>> cProfile.runctx('for i in range(5000):UserModelSerializer(u).data',globals(),locals(),sort="tottime")
         14540293 function calls (14220228 primitive calls) in 10.370 seconds

   Ordered by: internal time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
  6110003    1.073    0.000    1.073    0.000 {built-in method builtins.hasattr}
    65000    1.012    0.000    2.055    0.000 functional.py:125(__prepare_class__)
    50000    0.669    0.000    5.934    0.000 field_mapping.py:66(get_field_kwargs)
    55000    0.533    0.000    0.734    0.000 fields.py:309(__init__)
835004/835003    0.479    0.000    0.501    0.000 {built-in method builtins.getattr}
     5000    0.405    0.000    9.063    0.002 serializers.py:989(get_fields)
   210000    0.405    0.000    1.148    0.000 trans_real.py:274(gettext)
420000/210000    0.362    0.000    0.433    0.000 gettext.py:492(gettext)
    75000    0.321    0.000    2.733    0.000 functional.py:234(wrapper)
    20000    0.265    0.000    0.769    0.000 fields.py:769(__init__)
  1250012    0.248    0.000    0.297    0.000 {built-in method builtins.isinstance}
    50000    0.247    0.000    6.435    0.000 serializers.py:1194(build_standard_field)
                                              .
                                              .
                                              .
```
總共花費10.370秒，跟第一次的serializer的速度相差非常多  
接下來在python manage.py shell觀察一下ModelSerializer幫我們做了哪些事情  
```
>>> print(repr(UserModelSerializer()))
UserModelSerializer():
    id = IntegerField(label='ID', read_only=True)
    last_login = DateTimeField(allow_null=True, required=False)
    is_superuser = BooleanField(help_text='Designates that this user has all permissions without explicitly assigning them.', label='Superuser status', required=False)
    username = CharField(help_text='Required. 150 characters or fewer. Letters, digits and @/./+/-/_ only.', max_length=150, validators=[<django.contrib.auth.validators.UnicodeUsernameValidator object>, <UniqueValidator(queryset=User.objects.all())>])
    first_name = CharField(allow_blank=True, max_length=30, required=False)
    last_name = CharField(allow_blank=True, max_length=150, required=False)
    email = EmailField(allow_blank=True, label='Email address', max_length=254, required=False)
    is_staff = BooleanField(help_text='Designates whether the user can log into this admin site.', label='Staff status', required=False)
    is_active = BooleanField(help_text='Designates whether this user should be treated as active. Unselect this instead of deleting accounts.', label='Active', required=False)
    date_joined = DateTimeField(required=False)

```
最主要的參數是label、read_only、allow_null、required、help_text、allow_blank，以及在username的validators  
對於Core arguments可以參考官方文件 :   
https://www.django-rest-framework.org/api-guide/fields/    

其中也許是validators對效能造成了較大的影響  
因為在UniqueValidator的source code中，使用了gettext_lazy，而gettext_lazy則是使用了django.utils.functional中的lazy function  
lazy function : https://github.com/django/django/blob/master/django/utils/functional.py  
  
在lazy function中，使用了@total_ordering這個裝飾器，而在官方文件中寫到這個裝飾器會造成效能降低  
@total_ordering 官方文件 : https://docs.python.org/3.8/library/functools.html  

或許這就是造成ModelSerializer效能降低的問題
   
##  測試三  
因為ModelSerializer只會對writeable fields添加Field validations，所以接下來仍然使用ModelSerializer，但是要改成readonly  
```
class UserReadOnlyModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id','last_login','is_superuser','username','first_name','last_name','email','is_staff','is_active','date_joined']

        read_only_fields = fields
```
```
>>> cProfile.runctx('for i in range(5000):UserReadOnlyModelSerializer(u).data',globals(),locals(),sort="tottime")
         14975003 function calls (14675003 primitive calls) in 10.561 seconds

   Ordered by: internal time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
  6085000    1.092    0.000    1.092    0.000 {built-in method builtins.hasattr}
    65000    1.064    0.000    2.137    0.000 functional.py:125(__prepare_class__)
    50000    0.690    0.000    6.017    0.000 field_mapping.py:66(get_field_kwargs)
    55000    0.550    0.000    0.753    0.000 fields.py:309(__init__)
   835000    0.482    0.000    0.504    0.000 {built-in method builtins.getattr}
     5000    0.421    0.000    9.221    0.002 serializers.py:989(get_fields)
   210000    0.406    0.000    1.162    0.000 trans_real.py:274(gettext)
420000/210000    0.371    0.000    0.442    0.000 gettext.py:492(gettext)
    75000    0.263    0.000    2.796    0.000 functional.py:234(wrapper)
  1255000    0.250    0.000    0.296    0.000 {built-in method builtins.isinstance}
    50000    0.228    0.000    6.467    0.000 serializers.py:1194(build_standard_field)
    50000    0.214    0.000    0.360    0.000 serializers.py:1312(include_extra_kwargs)
                                              .
                                              .
                                              .
```
這次花費了10.561秒，和上一次沒有使用readonly的效能是差不多的  
原文的測試改成readonly之後，效能提升是非常明顯的，但是在測試三中的結果，卻沒有效能上的提升  
在原文中，使用的django和djangorestframework版本較舊，為Django 2.1.1和Django Rest Framework 3.9.4  
  
而這篇文章測試時使用的django和djangorestframework版本，或許已經將舊版本中的效能問題修正  
  
## 測試四
ModelSerializer因為使用了field_mapping.py也耗費了一定的效能資源  
所以接下來要測試一般的serializer  
```
class UserSerializer(serializers.Serializer):
    id = serializers.IntegerField()
    last_login = serializers.DateTimeField()
    is_superuser = serializers.BooleanField()
    username = serializers.CharField()
    first_name = serializers.CharField()
    last_name = serializers.CharField()
    email = serializers.EmailField()
    is_staff = serializers.BooleanField()
    is_active = serializers.BooleanField()
    date_joined = serializers.DateTimeField()
```
```
>>> cProfile.runctx('for i in range(5000):UserSerializer(u).data',globals(),locals(),sort="tottime")
         3250065 function calls (3150033 primitive calls) in 2.851 seconds

   Ordered by: internal time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
    55000    0.411    0.000    0.555    0.000 fields.py:309(__init__)
105000/5000    0.273    0.000    1.632    0.000 copy.py:128(deepcopy)
    50000    0.214    0.000    1.103    0.000 fields.py:633(__deepcopy__)
310003/310002    0.119    0.000    0.119    0.000 {built-in method builtins.getattr}
    20000    0.116    0.000    0.402    0.000 fields.py:769(__init__)
    50000    0.115    0.000    0.168    0.000 fields.py:355(bind)
     5000    0.103    0.000    2.639    0.001 serializers.py:504(to_representation)
    50000    0.083    0.000    0.170    0.000 fields.py:58(is_simple_callable)
   235000    0.081    0.000    0.081    0.000 {method 'update' of 'dict' objects}
   310000    0.077    0.000    0.117    0.000 {built-in method builtins.isinstance}
     5000    0.072    0.000    1.573    0.000 copy.py:257(_reconstruct)
    55000    0.066    0.000    0.082    0.000 fields.py:623(__new__)
     5000    0.058    0.000    0.179    0.000 fields.py:816(__init__)
    50000    0.054    0.000    0.307    0.000 fields.py:81(get_attribute)
    50000    0.054    0.000    0.222    0.000 serializer_helpers.py:143(__setitem__)
    55000    0.053    0.000    2.048    0.000 serializers.py:370(_readable_fields)
```
這次的測試僅僅使用了2.851秒，提升了非常多  
從這裡看來，捨棄ModelSerializer的便利性，選擇使用一般的Serializer對於效能表現會是非常好的決定  
  
##  測試五
在實作API的時候，仍然會有像是username，需要使用UniqueValidator驗證的資料  
所以這次使用測試四的UserSerializer，然後自己加上UniqueValidator  
```
class UserSerializer(serializers.Serializer):
    id = serializers.IntegerField()
    last_login = serializers.DateTimeField()
    is_superuser = serializers.BooleanField()
    username = serializers.CharField(validators=[UniqueValidator(queryset=User.objects.all())])
    first_name = serializers.CharField()
    last_name = serializers.CharField()
    email = serializers.EmailField()
    is_staff = serializers.BooleanField()
    is_active = serializers.BooleanField()
    date_joined = serializers.DateTimeField()
```
```
>>> cProfile.runctx('for i in range(5000):UserSerializer(u).data',globals(),locals(),sort="tottime")
         3250065 function calls (3150033 primitive calls) in 2.859 seconds

   Ordered by: internal time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
    55000    0.444    0.000    0.585    0.000 fields.py:309(__init__)
105000/5000    0.253    0.000    1.630    0.000 copy.py:128(deepcopy)
    50000    0.232    0.000    1.125    0.000 fields.py:633(__deepcopy__)
310003/310002    0.118    0.000    0.118    0.000 {built-in method builtins.getattr}
    50000    0.117    0.000    0.166    0.000 fields.py:355(bind)
    20000    0.116    0.000    0.419    0.000 fields.py:769(__init__)
     5000    0.103    0.000    2.647    0.001 serializers.py:504(to_representation)
    50000    0.082    0.000    0.172    0.000 fields.py:58(is_simple_callable)
   235000    0.076    0.000    0.076    0.000 {method 'update' of 'dict' objects}
     5000    0.068    0.000    1.569    0.000 copy.py:257(_reconstruct)
   310000    0.066    0.000    0.106    0.000 {built-in method builtins.isinstance}
    55000    0.065    0.000    0.080    0.000 fields.py:623(__new__)
    50000    0.063    0.000    0.316    0.000 fields.py:81(get_attribute)
    55000    0.055    0.000    0.076    0.000 copy.py:241(_keep_alive)
    55000    0.054    0.000    2.047    0.000 serializers.py:370(_readable_fields)
    50000    0.051    0.000    0.217    0.000 serializer_helpers.py:143(__setitem__)
     5000    0.046    0.000    1.912    0.000 serializers.py:351(fields)
                                              .
                                              .
                                              .
```
花費2.859秒  
看來就算加上validators，也不會對效能造成太大的負擔  
  
#  測試總結
因為版本的不同，效能差異也非常明顯  
在原文中較舊的版本，測試出來的效能與本文使用的django、djangorestframework版本測試出來的效能差異非常明顯   
  
不使用ModelSerializer，改成使用需要自己寫fields的Serializer可以提升非常多效能  

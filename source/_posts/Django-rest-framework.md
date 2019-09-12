---
title: 'Django 與 REST API 碰撞的火花' 
date: 2017-07-13 14:55:39
tags: 實習
---
剛好在實習時接觸到 Django 和 REST API 的概念，所以就來整理跟分享與 Django REST Framework 有關的部分吧！如果有任何想法或是疑問，都可以再討論喔！

## REST API
先來說說 REST API，有關詳細解釋可以參考維基百科的[說明頁面](https://zh.wikipedia.org/zh-tw/REST)。這邊我就僅僅提出部分重點，可以與後面的部分互相對應。

主要提出四點：

- 客戶端和伺服器結構：自從行動裝置的盛行，伺服器端必須提供給客戶端對資源進行操作的管道。
- 連接協議具有無狀態性：<!--more-->以開發客戶端的角度，必須了解每次的請求的狀態如何，才容易知道是甚麼環節出錯，諸如欄位驗證，伺服器狀態，認證狀態等。
- 一致性的操作界面：這邊的介面傾向於 CRUD 對應 GET、POST、UPDATE、DELETE 的請求差異，搭配制式格式欄位，會讓需要使用的開發者清楚如何操作。
- 層次化的系統：其實和上面十分相似，例如取得所有資源或是單項資源。

這些會和接下來的 Django REST Framework 某些模組有相互對應的感覺。

## Django REST Framework
在 Python 中有許多有關的模組，能協助開發者以更簡便的方式開發 API，舉凡較常見的 Flask、Pyramid。如果純粹開發 API 或許看個人習慣而有所差異，但如果同時顧及 Django 在 MVC 方面的開發，加上既有資料 models 方面，如此來說就可以擁有事半功倍的效果，那麼 Django REST Framework 會是一個不錯的選擇。

關於 Django 的部分，由於此重點不再此，所以等待後續有機會再另行介紹。還有如果想嘗試，也適合自己

從[官方 Tutorial](http://www.django-rest-framework.org/#tutorial) 的 Quick Start 部分，可以發現幾個比較重要的概念和模組。例如 Serialization、Request & Responses、Viewsets、權限、授權等相關類別，當然還有 Filtering、Throtting、Pagination 等重要概念未在 Tutorial 中。底下就稍微介紹各種用法吧！

### Serialization
在原本 Django 在資料 models 的規劃就有一定做法，以建立型別，如：
```.py
class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)
```
這邊的 Class 有些類似一般資料庫的 Table 的韻味，至於相關型別規劃則繼承自 Django 的 Model 模組，然而在 Serializer 這邊則是引用 REST Framework 的 serializers。範例如下：
```.py
class SnippetSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')
```
除了 import 的來源不同以外，可以使用的基本型別也變多了。大家所好奇的是，如果僅僅型別差異，又為何需要使用這些東西，這些其實也都能從最基本型別中自造出來。最主要的差別在於，透過如此型別規劃，可以避免掉自己解析客戶端送來的原始資料，用少量的 code 做完同樣的事，同時往後的增刪改查等動作也能直接複寫既有的方法進行處理。如：
```.py
    def create(self, validated_data):
        """
        Create and return a new `Snippet` instance, given the validated data.
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        Update and return an existing `Snippet` instance, given the validated data.
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```

### Viewset
其實 Viewset 就有提供剛剛說的功能，但是不提供與 Request 相近的方法，如 GET，POST，但還是有 list retrieve create 等較進階作法，不過因為是 Class-based View，所以只要繼承原先設計好的 viewsetd.ViewSet，再覆寫，就可使用。相關案例如下：
```
from django.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from myapps.serializers import UserSerializer
from rest_framework import viewsets
from rest_framework.response import Response

class UserViewSet(viewsets.ViewSet):
    """
    A simple ViewSet for listing or retrieving users.
    """
    def list(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        queryset = User.objects.all()
        user = get_object_or_404(queryset, pk=pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)
```
後頭再把這兩個 View 區分，連到 Router 部分，即可完成簡易設定，而這就等同於用原有 View 的概念完成相關設定。

### 其他
比如說 Filter 就是提供條件篩選的方式提取資料，如果沒有 Framework，可能會需要針對不同情況進行解析及邏輯處理，只要多一種狀況可能性就會多出許多需要處理的 Filter，套用 Framework，讓你避免不必要的做事內容。

Thortting 能輕鬆做出限制單位時間的流量或連線數，這如果是進行商業模式會是受限於主機成本，都能有效控管，使部署應用時，變得比較方便。

Paginaton 則是在提供再提取資料時，若一次筆數過多，必須提供分段提取，而客戶端所收到的內容大致會有該次最大筆提取內容，以及下次提取內容時的 URL 或是 Token/Tag (依設計而定)，在下一次提取時，會有上一次及下一次的連結。如果不知道具體行為如何，可以去玩玩看 Facebook 的 Graph API，提取自己的動態時報資料，就能理解了。以 JSON 格式來看，大致如下：
```json
{
	status:200,
	content:[
		{...},
		{...},
		...
		{...}
	],
	link:{
		next_page:'http://example.com/page?tag=jfksdlfjlskdjflskdjfklsdj'
	}
}
```

## 小記
其實這塊也是初次碰觸，如果之後對某些套件有更深的體悟時，或許會有針對某些套件的心得。最後當然老話一句，如果有任何想法，歡迎糾錯、切磋。
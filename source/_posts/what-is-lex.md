---
title: Lex 超新手入門
tags: compiler
date: 2017-04-26 01:30:43
---

因為系上 Compiler 課程上到 Lex & Yacc，但是不論講義還是網路上資源都僅僅皮毛而已，總覺得搔不到邊，只好自己借歐禮萊的書自己 K。至於這篇文章就當作是閱讀筆記吧，希望能造福更多被 Lex 荼毒的眾生，至於 Yacc 就等後集待續吧！
<!--more-->
## Regular Expression
在搞懂 Lex 是如何運作前，須先了解 Regular Expression (正規表示式) 是啥？Reglar Expression (以下簡稱 RE) 是種規則敘述，透過定義，能協助我們分辨、歸類。而 Lex 在 token 的定義就是透過 RE 去實作。以下用簡表來提及一些基本的 RE 特殊字元。

|Character(s)|Meaning|
|:---|:---|
| * | 包含**零個**到**無限多個** |
| + | 包含**一個**到**無限多個** |
| ? | 包含**零個**或**一個** |
| [] | 出現集合，在集合中擇一，如果是數字或大小寫英文的連續區間，請用 **「-」** 隔開|
| {} | 設定重複次數，如果是一個區間，用**逗號**隔開|
| () | 群組的概念，裏頭可接其他 RE |
| \ | 跳脫字元，當用到如小數點的狀況，這樣才能把字元挑出來，還有其他如換行、tab |
| ^ | 當在 **[]** 內時，表示**排除**；當在外面時，表示為**行尾** |
| $ | 表示**行首**|
| \|  通常表示字詞擇一 |

其實還有一些進階而且強大的功能沒有講到，如果有興趣或需要者可以留言，我再回應或增加。
同時在這邊提供一些檢測是否選取成功的線上工具。
regular expressions 101: [https://regex101.com/](https://regex101.com/)
RegExr : [http://regexr.com/](http://regexr.com/)

## Lex
基本上，Lex 可先分為三大區塊，各大區塊以 <code>%%</code> 分開，分別是 *definition*、*rule*、*user subroutines*。
在程式中所呈現的方式如下：
```
	definition
%%
	rule
%%
	user subroutines
```
接下來分各部分介紹。

### Definition Section
這部分主要是以宣告替換字詞或 C code 為主，但有一些注意事項：

1. C code 是用來宣告變數或是 include 非基本 C Header File，並且需要以 <code>%{</code> 和 <code>%}</code> 包起來
2. 其他替代字詞需要在 <code>%{</code><code>%}</code> 外面宣告
3. 在替代字詞有分成兩塊，一部分是需替代的字詞，一部份是替換後字詞，兩個之間要用多個空白分開
4. 還有一些比較神奇的做法，一個替換一個，用組合的方式處理
```lex
%{
	#include<string.h>
	/*
		You don't have to include stdlib.h
	*/
%}
num  ([0-9]|[1-9][0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-5])
dot  \.
ip  {num}{dot}{num}{dot}{num}{dot}{num}
%%
...
%%
...
```

### Rule Section
這部分主要在設定 RE 與相對應動作的部分，當然也可以透過前面定義好的東西進行代換，建議是將 token 設得較小，避免篩不到想要的字詞。在輸出方面，常見當然是把剛剛選取完的字詞進行輸出，而方法有二，ECHO 和 yytext。
其實 yytext 就是一個 string buffer 儲存目前所篩出來的字詞，而 ECHO 就把 printf 一同包進去了，但是請注意，僅限目前字詞，若目前字詞沒有換行，他們就不會換行喔！
轉換成 code 來說，就是 <code>ECHO;</code> 相等於 <code>printf("%s",yytext);</code>
最後幾點提醒：
1. RE 與相對應動作的 code 之間需用**兩個以上**的空格分開。
2. 相對應動作的 code 需用 <code>{}</code> 包起來，除非無動作，才可以以 <code>{;}</code> 或 <code>;</code> 表示
3. C code 結束要以<code>;</code>結尾！要以<code>;</code>結尾！要以<code>;</code>結尾！
```lex
%{
	#include<string.h>
	/*
		You don't have to include stdlib.h
	*/
}%
num  ([0-9]|[1-9][0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-5])
dot  \.
ip  {num}{dot}{num}{dot}{num}{dot}{num}
%%
{ip}  {printf("%s\n",yytext);}
\n  ;
.   ;
%%
...
```

### User Subroutines Section
簡單來說就是 Main Function 啦！如果你前兩個 section 處理得不錯，那代表這裡只需要用最單純的部分，
```lex
int main()
{
	yylex();
	return(0);
}
```
但其實 Lex 還是有提供 yywrap() 的 function 自定義，只是究竟該如何使用還沒搞清楚，如果搞清楚了再告訴各位。

## Lex 編譯 & 執行
這邊先假設大家都是在 Unix 的環境下，已經有 gcc 及 flex，可以進行處理，如果還沒，請左轉出去問問 Google 大神。
我的習慣會準備一個輸入檔，接著假設你的 Lex file 是 test.l，輸入檔是 input.txt。
開啟命令列，打入
```bash
flex test.l
```
接著輸入
```bash
gcc yy.lex.c -o test
```
目錄底下就會有一個叫做 test 的執行檔了。
接下來把輸入檔餵進去，
```bash
./test < input.txt
```
Terminal 上應該就可以看到有相對應結果顯示。

這篇也先到這裡，如果有問題歡迎發問喔！或是有誤也歡迎更正！

## 參考書目
lex & yacc, O'REILLY, 2000
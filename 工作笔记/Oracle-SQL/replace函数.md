## replace函数

```sql
replace(string,from_str,to_str)
/*
eg.select replace(a.insure_no,'-','')
```

## trim函数

去掉首尾空格，一个参数

## decode函数

```sql
select decode(columnname，值1,翻译值1,值2,翻译值2,...值n,翻译值n,缺省值)
```

现定义一table名为output，其中定义两个column分别为monthid（var型）和sale（number型），若sale值=1000时翻译为D，=2000时翻译为C，=3000时翻译为B，=4000时翻译为A，如是其他值则翻译为Other：

```sql
select monthid , decode (sale,1000,'D',2000,'C',3000,'B',4000,'A',’Other’) sale from output
```

## LPAD函数

```sql
LPAD(str,len,padstr)
```

返回字符串str，左填充len个字符长度的字符串padstr。若str的长度大于len，则返回值被缩短至len个字符。

## 字符串连接符 ||

## substr(str,start,lenth)

从0和1开始截取是一样的


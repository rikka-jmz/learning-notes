# PL/SQL编程简介

* 块结构
* 条件逻辑和循环
* 游标
* 存储过程和函数

## 块结构

oracle中引入的一种过程化编程语言，称为PL/SQL。可划分为称为块的结构。PL/SQL中的块分为三种：匿名块、函数、存储过程。这里的块讲的是匿名块。

典型的PL/SQL代码块包含如下结构：

```plsql
[DECLARE
	declaration_statements     -- 声明PL/SQL块其余部分使用的变量。DECLARE块是可选的
]
BEGIN
	executable_statements      -- 块中实际可执行的语句，其中可能包括执行诸如循环、条件逻辑等任务语句
[EXCEPTION
    exception_handing_statements -- 负责处理当块运行时可能发生的任何可执行错误。EXCEPTION块是可选的。
]
END;  -- 每一条语句都以分号结尾。
/     -- PL/SQL块使用正斜杠字符结尾

-- example --
DECLARE                -- 变量在DECLARE块中声明，变量声明中同时包含名称和类型，可以赋默认值。
-- declare用来声明变量，格式为：变量名 变量类型（数据长度）NOT NULL:= 默认值
	v_width INTEGER;   
	v_height INTEGER :=2;-- :=默认值
	v_area INTEGER :=6;  --？
BEGIN
	v_width :=v_area/v_height; -- :=为赋值符号
	DBMS_OUTPUT.PUT_LINE('v_width='||v_width); -- DBMS_OUTPUT.PUT_LINE()为内置输出语句，||为字符串连接符
EXCEPTION
	WHEN ZERO_DEVIDE THEN
		DBMS_OUTPUT.PUTLINE('Division by zero');
END;
/
```

## 条件逻辑和循环

### 条件逻辑

```plsql
IF condition1 THEN
	statements1
ELSEIF condition2 THEN
	statements2
……
ELSE 
	statements3
END IF
```

### 循环

循环有三种类型

* **简单循环：**直到显式结束循环之前一直运行

  ```plsql
  LOOP
  	statements
  END LOOP
  /*
  1.要结束简单循环，可以使用EXIT或EXIT WHEN语句
  2.可以使用CONTINUE或CONTINUE WHEN语句循环当前的迭代
  */
  ```

* **while循环：**直到某个特定条件出现之前一直运行

  ```plsql
  WHILE condition LOOP
  	statements
  END LOOP;
  --example--
  -- v_counter<6时会一直执行
  v_counter :=0;
  WHILE v_counter<6 LOOP
  	v_counter:=v_counter+1;
  END LOOP;
  ```

* **for循环：**运行预先确定的次数

  ```plsql
  FOR loop_variable IN [PEVERSE] lower_bound ..upper_bound LOOP statements
  END LOOP
  -- loop_variable:指定循环变量。循环变量的值每次循环自增1，若使用[PEVERSE]关键字则每次递减1
  -- lower_bound:循环下限 upper_bound:循环上限
  -- example --
  FOR v_counter2 IN 1..5 LOOP
  	DBMS_OUTPUT.PUT_LINE(v_counter2);
  END LOOP;
  ```

  

## 游标

### 概念

用来处理从数据库中检索出的多行记录（使用select语句）。利用游标，程序可以逐个处理和遍历一次检索返回的整个数据集（存放在内存）

### 分类

* 静态游标：编译时知道其select语句的游标。分为隐式游标和显式游标。
* 动态游标：用户为游标使用的查询知道运行时才能确定。可以使用REF游标和游标变量满足这个要求。为了使用引用游标，必须声明游标变量。两种类型REF游标：强类型REF游标和弱类型REF游标。

### 静态游标_显式游标

显式游标多被用于处理返回多行数据的select语句，游标名通过cursor···.IS语句显式的赋给select语句。

```
使用步骤：
1、声明游标 CURSOR cursor_name IS select_statement
2、为查询打开游标：OPEN cursor_name
3、取得结果放入PL/SQL变量中
	FETCH cursor_name INTO list_of_variables;
	FETCH cursor_name INTO PL/SQL_record;
4、关闭游标。CLOSE cursor_name
注意：在声明游标时，select_statement不能包含INTO子句。当使用显式游标时，INTO子句是FETCH语句的一部分
```

```plsql
--exampl01 使用LOOP遍历游标--
DECLARE 
	v_name emp.ename%TYPE;  -- %TYPE 与数据库列相同数据类型
                            -- %ROWTYPE 与数据库行相同数据类型
	v_sal emp.sal%TYPE;
	CURSOR cus_emp IS
		SELECT ename,sal from emp; -- 声明游标
BEGIN
	OPEN cus_emp;                  -- 打开游标
	LOOP
		FETCH cus_emp INTO v_name,v_sal; -- 提取游标
		EXIT WHEN cus_emp%NOTFOUND;
		/*
		  显式游标的四个属性：
		  游标变量 %found: 当最近一次读入记录成功时返回true
		  游标变量 %notfound:同上 相反
          游标变量 %isopen:判断游标是否已经打开
          游标变量 %rowcount:返回已从游标中读取的记录数
        */
		DBMS_OUTPUT.PUT_LINE('第'||cus_emp%ROWCOUNT||'个用户：name：'||v_name||'	sal:'||v_sal);
	END LOOP;
	CLOSE cus_emp; -- 关闭游标
END;
```

```plsql
--exampl02 使用for遍历游标--
DECLARE 
	CURSOR cus_emp IS
		SELECT ename,sal FROM emp;
BEGIN
	FOR record_emp IN cus_emp
	LOOP
		DBMS_OUTPUT.PUT_LINE('第'||cus_emp%ROWCOUNT||'个用户：name：'||recored_emp.ename||'	sal:'||record_emp.sal);
	END LOOP;
END;
-- 循环游标隐式打开游标，自动创建临时记录类型变量存储记录，处理完后自动关闭游标。
```

### 静态游标_隐式游标

* 所有隐式游标都被假设为只返回一条记录
* 使用隐式游标时，用户无需进行声明、打开及关闭。PL/SQL隐式的打开、处理然后关闭游标。
* 多条sql语句，隐式游标永远指的是最后一条sql语句的结果，主要使用在update和delete语句上。

四个属性：

```
* SQL%rowcount：影响的记录行数整数
* SQL%found：影响到了记录 true()
* SQL%notfound：没有影响到记录true()
* SQL%isopen：是否打开布尔值，永远是false
```

示例：

```plsql
DECLARE 
	row_emp emp%ROWTYPE;
BEGIN
	SELECT ename,sal INTO row_emp.ename,row_emp.sal
	FROM emp WHERE emp.empno=7369;
	/*
	SELECT INTO 语句从一个表中选取数据，然后把数据插入另一个表中。
    SELECT INTO 语句常用于创建表的备份复件或者用于对记录进行存档。
    语法:
    SELECT *
    INTO new_table_name [IN externaldatabase] 
    FROM old_tablename
    */
    -- 判断是否查到数据
    IF(SQL%ROWCOUNT=1) THEN
    	DBMS_OUTPUT.PUT_LINE('找到了');
    END IF;
    -- 另一种方式判断
    IF(SQL%FOUND) THEN
    	DBMS_OUTPUT.PUT_LINE('找到了');
    END IF;
    
    DBMS_OUTPUT.PUT_LINE('ename:'||row_emp.ename||'	sal:'||row_emp.sal);
END;
-- 上述游标自动打开，并把相关值赋给对应变量，然后关闭。执行完成后，PL/SQL变量rowemp.ename,rowemp.sal中已经有了值。
```

## 存储过程和函数

* 相同：完成特定功能的程序
* 不同：是否用return语句返回值

### 存储过程

存储在数据库中提供所有用户程序调用的子程序。关键字**procedure**。

创建存储过程：

```plsql
CREATE [OR REPLACE] PROCEDURE 存储过程名
	[(参数[IN|OUT|IN OUT] 数据类型...)]
{AS|IS}
	[说明部分]
BEGIN
	[可执行部分]
[EXCEPTION
	错误处理部分]
END [过程名];
-- 可选关键字REPLACE表示如果存储过程已经存在，则用新的存储过程覆盖，通常用于存储过程的重建
-- 参数部分用于定义多个参数（如果没有参数，就可以省略）。参数有三种形式IN、OUT、IN OUT。如果没有指明参数形式，则默认为IN
-- 关键字AS也可以写成IS，后跟过程的说明部分，可以在此定义过程的局部变量。
-- 编译成功的存储过程可以直接在Oracle环境下进行调用。
```

执行（调用）存储过程：

```plsql
-- 方法一
	EXECUTE 模式名.存储过程名[(参数...)];
-- 方法二
	BEGIN
		模式名.存储过程名[(参数...)];
	END;
-- 说明
-- 1.传递的参数必须与定义的参数类型、个数和顺序一致。
-- 2.调用本账户的存储过程时，模式名可以省略。
```

### 函数

* 函数必须有```return```子句，定义关键词为```function```。

创建函数：

```PLSQL
CREATE [OR REPLACE] FUNCTION 函数名
	[(参数[IN] 数据类型...)]
RETURN 数据类型
{AS|IS}
	[说明部分]
BEGIN
	[可执行部分]
	RETURN (表达式)
[EXCEPTION
	错误处理部分]
END [函数名];

--参数可选，但只能是IN类型，IN关键字可以省略
--在定义部分的RETURN数据类型，表示函数的数据类型，即返回类型。
--可执行部分的RETURN用来生成函数的返回值。
```




---
abbrlink: ''
categories:
- - 单片机
date: '2023-12-11T21:51:53.568128+08:00'
tags:
- STM32
title: Usmart
updated: '2023-12-17T23:10:11.094+08:00'
---
# Usamrt流程

## 头文件内容解析

+ usmart.h----usmart的主要函数

```c
//8个函数函数
void usmart_init(u8 sysclk);		// 初始化
u8 usmart_cmd_rec(u8 *str);			// 识别
void usmart_exe(void);				// 执行
void usmart_scan(void);				// 扫描
u32 read_addr(u32 addr);			// 读取指定地址的值
void write_addr(u32 addr, u32 val); // 在指定地址写入指定的值
u32 usmart_get_runtime(void);		// 获取运行时间
void usmart_reset_runtime(void);	// 复位运行时间
//两个结构体
// 函数名列表
struct _m_usmart_nametab
{
	void *func;		// 函数指针
	const u8 *name; // 函数名(查找串)
};
// usmart控制管理器
struct _m_usmart_dev
{
	struct _m_usmart_nametab *funs; // 函数名指针

	void (*init)(u8);		// 初始化
	u8 (*cmd_rec)(u8 *str); // 识别函数名及参数
	void (*exe)(void);		// 执行
	void (*scan)(void);		// 扫描
	u8 fnum;				// 函数数量
	u8 pnum;				// 参数数量
	u8 id;					// 函数id
	u8 sptype;				// 参数显示类型(非字符串参数):0,10进制;1,16进制;
	u16 parmtype;			// 参数的类型
	u8 plentbl[MAX_PARM];	// 每个参数的长度暂存表
	u8 parm[PARM_LEN];		// 函数的参数
	u8 runtimeflag;			// 0,不统计函数执行时间;1,统计函数执行时间,注意:此功能必须在USMART_ENTIMX_SCAN使能的时候,才有用
	u32 runtime;			// 运行时间,单位:0.1ms,最大延时时间为定时器CNT值的2倍*0.1ms
};
//一些变量定义不需要解释
```

+ usmart_str.h----usmart字符串的相关操作

```c
u8 usmart_get_parmpos(u8 num);                                    // 得到某个参数在参数列里面的起始位置
u8 usmart_strcmp(u8 *str1, u8 *str2);                             // 对比两个字符串是否相等
u32 usmart_pow(u8 m, u8 n);                                       // M^N次方
u8 usmart_str2num(u8 *str, u32 *res);                             // 字符串转为数字
u8 usmart_get_cmdname(u8 *str, u8 *cmdname, u8 *nlen, u8 maxlen); // 从str中得到指令名,并返回指令长度
u8 usmart_get_fname(u8 *str, u8 *fname, u8 *pnum, u8 *rval);      // 从str中得到函数名
u8 usmart_get_aparm(u8 *str, u8 *fparm, u8 *ptype);               // 从str中得到一个函数参数
u8 usmart_get_fparam(u8 *str, u8 *parn);                          // 得到str中所有的函数参数.
```

### u8 usmart_get_fname

> pcnt & 0x7f 应该表示第几个参数
> strtemp每次循环指向一个字符
> 首先判断接收到的字符串前五个字符是不是void, 设置rval, 是不是需要返回值
> 注意! 接收完前五个字符, 记得加'\0'  字符串结束标记, 否则usmart_strcmp无法比较
> 下一个while判断当前字符是不是'*'或者'(' , 判断逻辑是, 没有碰到空格或者 * 就继续循环, 同时res也就是offset加1
> 根据offset直接跳到函数名开始的地方
> 继续判断, 逻辑是   fover表示碰到的括号, 左括号加一, 右括号减一,  如果fover是0说明没有接收完, 因为至少有一个左括号或有右括号
> 判断是不是','   是,temp加一(temp表示参数个数)

```c
// 从str中得到函数名
//*str:源字符串指针
//*fname:获取到的函数名字指针
//*pnum:函数的参数个数
//*rval:是否需要显示返回值(0,不需要;1,需要)
// 返回值:0,成功;其他,错误代码
u8 usmart_get_fname(u8 *str, u8 *fname, u8 *pnum, u8 *rval)
{
	u8 res;
	u8 fover = 0; // 括号深度
	u8 *strtemp;
	u8 offset = 0;
	u8 parmnum = 0;
	u8 temp = 1;
	u8 fpname[6];  // void+X+'/0'
	u8 fplcnt = 0; // 第一个参数的长度计数器
	u8 pcnt = 0;   // 参数计数器
	u8 nchar;
	// 判断函数是否有返回值
	strtemp = str;
	while (*strtemp != '\0') // 没有结束
	{
		if (*strtemp != ' ' && (pcnt & 0X7F) < 5) // 最多记录5个字符
		{
			if (pcnt == 0)
				pcnt |= 0X80; // 置位最高位,标记开始接收返回值类型   x000 0x00
			if (((pcnt & 0x7f) == 4) && (*strtemp != '*'))  // 0x7f ---> 0111 1111   和0x7f相与查看pcnt是不是等于4    且strtemp此时指向的不是 * 
				break;						// 就跳出当前这一次循环, 因为最后一个字符,必须是*
			fpname[pcnt & 0x7f] = *strtemp; // 记录函数的返回值类型   TODO: ?????
			pcnt++;
		}
		else if (pcnt == 0X85)// 0x85 ---> 1000 0101
			break;
		strtemp++;
	}
	if (pcnt) // 接收完了
	{
		fpname[pcnt & 0x7f] = '\0'; // 加入结束符
		if (usmart_strcmp(fpname, "void") == 0) // 看是不是void
			*rval = 0; // 不需要返回值
		else
			*rval = 1; // 需要返回值
		pcnt = 0;
	}
	res = 0;
	strtemp = str;
	while (*strtemp != '(' && *strtemp != '\0') // 此代码找到函数名的真正起始位置
	{
		strtemp++;
		res++;
		if (*strtemp == ' ' || *strtemp == '*')
		{
			nchar = usmart_search_nextc(strtemp); // 获取下一个字符
			if (nchar != '(' && nchar != '*')
				offset = res; // 跳过空格和*号
		}
	}
	strtemp = str;
	if (offset)
		strtemp += offset + 1; // 跳到函数名开始的地方
	res = 0;
	nchar = 0; // 是否正在字符串里面的标志,0，不在字符串;1，在字符串;
	while (1)
	{
		if (*strtemp == 0)
		{
			res = USMART_FUNCERR; // 函数错误
			break;
		}
		else if (*strtemp == '(' && nchar == 0)
			fover++; // 括号深度增加一级
		else if (*strtemp == ')' && nchar == 0)
		{
			if (fover)
				fover--;
			else
				res = USMART_FUNCERR; // 错误结束,没收到'('
			if (fover == 0)
				break; // 到末尾了,退出
		}
		else if (*strtemp == '"')
			nchar = !nchar;

		if (fover == 0)  // 已经接受完了函数名了
		{
			if (*strtemp != ' ') // 空格不属于函数名
			{
				*fname = *strtemp; // 得到函数名
				fname++;
			}
		}
		else // 函数名还没接收完
		{
			if (*strtemp == ',')
			{
				temp = 1; // 使能增加一个参数
				pcnt++;
			}
			else if (*strtemp != ' ' && *strtemp != '(')
			{
				if (pcnt == 0 && fplcnt < 5) // 当第一个参数来时,为了避免统计void类型的参数,必须做判断.  void(超过5了,所以定为5
				{
					fpname[fplcnt] = *strtemp; // 记录参数特征
					fplcnt++;
				}
				temp++; // 得到有效参数(非空格)
			}
			if (fover == 1 && temp == 2) // TODO:没懂？？？
			{
				temp++;	   // 防止重复增加
				parmnum++; // 参数增加一个
			}
		}
		strtemp++;
	}
	if (parmnum == 1) // 只有1个参数.
	{
		fpname[fplcnt] = '\0'; // 加入结束符
		if (usmart_strcmp(fpname, "void") == 0)
			parmnum = 0; // 参数为void,表示没有参数.
	}
	*pnum = parmnum; // 记录参数个数
	*fname = '\0';	 // 加入结束符
	return res;		 // 返回执行结果
}
```

### u8 usmart_strcmp(u8 *str1, u8 *str2)

> 比较两个字符串是不是一样, 和strcmp类似

```c
u8 usmart_strcmp(u8 *str1, u8 *str2)
{
	while (1)
	{
		if (*str1 != *str2)
			return 1; // 不相等
		if (*str1 == '\0')
			break; // 对比完成了.
		str1++;
		str2++;
	}
	return 0; // 两个字符串相等
}
```

### u8 usmart_search_nextc(u8 *str)

```c
// 获取下一个字符（当中间有很多空格的时候，此函数直接忽略空格，找到空格之后的第一个字符）
// str:字符串指针
// 返回值:下一个字符
u8 usmart_search_nextc(u8 *str)
{
	str++;
	while (*str == ' ' && str != '\0')
		str++;
	return *str;
}
```

### usmart_init----初始化

> 初始化定时器, 使用的是定时器4, 主频也就是sysclk是72mhz, sysclk*100-1是分频系数---> 72000000/(72 * 100-1 +1 )=10000hz, 所以一个定时器周期是1/10000==0.0001s==0.1ms, 定时器0.1ms*1000====100ms中断一次
> 
> ```c
> Timer4_Init(1000, (u32)sysclk * 100 - 1);
> ```

### usmart_cmd_rec----识别

> usmart_cmd_rec函数逻辑
> 入口参数是从串口接收到的函数名字, 字符串指针
> 首先通过usmart_get_fname得到函数名字和参数个数
> 然后从0到usmart_dev.fnum(结构体里函数数量值)开始循环, strcmp逐个比较接收的函数名字和本地存储的函数名字
> 情况1. 如果找到一样的，判断收到的函数参数是不是和存储的函数参数个数相等
> 情况1.1 收到的小于存储的就返回错误**USMART_PARMERR**, 记下函数id, 跳出循环
> 情况1.2 收到的大于存储的,     xxxxxxxxxxx
> 情况1.3 收到的等于存储的, 用usmart_get_fparam获得函数参数个数并赋值给usmart_dev.pnum
> 情况3. 如果i循环到等于usmart_dev.fnum, 也就是全部找完了, 没有发现符合的函数名字,也返回一个错误**USMART_NOFUNCFIND**
> 都判断完,找到了且参数个数也一样就返回**USMART_OK**

```c
//入口参数是字符串指针
u8 usmart_cmd_rec(u8 *str)
{
	u8 sta, i, rval; // 状态
	u8 rpnum, spnum;
	u8 rfname[MAX_FNAME_LEN];							// 暂存空间,用于存放接收到的函数名
	u8 sfname[MAX_FNAME_LEN];							// 存放本地函数名
	sta = usmart_get_fname(str, rfname, &rpnum, &rval); // 得到接收到的数据的函数名及参数个数
	if (sta)
		return sta; // 错误
	for (i = 0; i < usmart_dev.fnum; i++)
	{
		sta = usmart_get_fname((u8 *)usmart_dev.funs[i].name, sfname, &spnum, &rval); // 得到本地函数名及参数个数
		if (sta)
			return sta;							// 本地解析有误
		if (usmart_strcmp(sfname, rfname) == 0) // 相等
		{
			if (spnum > rpnum)
				return USMART_PARMERR; // 参数错误(输入参数比源函数参数少)
			usmart_dev.id = i;		   // 记录函数ID.
			break;					   // 跳出.
		}
	}
	if (i == usmart_dev.fnum)
		return USMART_NOFUNCFIND;	  // 未找到匹配的函数
	sta = usmart_get_fparam(str, &i); // 得到函数参数个数
	if (sta)
		return sta;		 // 返回错误
	usmart_dev.pnum = i; // 参数个数记录
	return USMART_OK;
}
// 从str中得到函数名
//*str:源字符串指针
//*fname:获取到的函数名字指针
//*pnum:函数的参数个数
//*rval:是否需要显示返回值(0,不需要;1,需要)
// 返回值:0,成功;其他,错误代码
u8 usmart_get_fname(u8 *str, u8 *fname, u8 *pnum, u8 *rval)
```

### usmart_exe----执行

### usmart_scan----扫描

### 整体逻辑

1. 初始化定时器四, 0.1ms产生一次中断, 执行usmart_scan进行扫描
2. 扫描成功后执行usmart_cmd_rec识别接收的字符串
3. 识别成功后执行usmart_exe运行函数


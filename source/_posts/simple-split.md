---
title: c语言一种简单可行的GBK编码的字符串的分割方法
date: 2022-11-09 10:01:07
tags: 杂文
categories: 杂文
---

该方法主要借鉴了编译原理中词法分析器的实现，根据gbk中汉字编码特征来进行状态转换。

```c
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>

enum STATE
{
	START_STATE, //开始状态
	CHINESE_STATE, //中文状态
	CHAR_STATE, //正常字符串状态
	COMP_SEP_STATE, //比较分隔符状态
	DONE_STATE //结束状态
};

// uint8_t中以0为结尾字符串长度读取
size_t
strlen(const uint8_t *str)
{
	size_t len = 0;
	while (str[len] != 0)
	{
		len++;
	}
	return len;
}

// char中以'\0'为结尾字符串长度读取（包括'\0'）；你要问为什么不用string.h的方法，因为任性！
size_t
charlen(const char *str)
{
	size_t len = 0;
	while (str[len] != '\0')
	{
		len++;
	}
	return ++len;
}

// uint8_t *拷贝到char *
void
strcpy(const uint8_t *src, char *dst)
{
	size_t len = strlen(src);
	for (size_t i = 0; i < len; i++)
	{
		dst[i] = src[i];
	}
	dst[len] = '\0';
}

// char *转换为uint8_t *
void convert(const char *src, uint8_t *dst)
{
	size_t len = charlen(src);
	for (size_t i = 0; i < len; i++)
	{
		dst[i] = src[i];
	}
}

/**
* 获取下一个子字符串
* 该方法主要通过状态转换机实现，当state转变为DONE_STATE时代表截取完成，具体不同状态间的转换规则自己参悟代码。
*/
uint8_t *
get_next(uint8_t *str, uint8_t *separator, size_t *pos)
{
	uint8_t *token = (uint8_t *)malloc(1000 * sizeof(uint8_t));
	token[0] = 0;
	size_t separator_pos = 0;
	size_t token_pos = 0;
	uint8_t chinese_flag = 0x81;
	enum STATE state = START_STATE;

	if (*pos >= strlen(str))
	{
		return NULL;
	}

	while (state != DONE_STATE)
	{
		if (*pos >= strlen(str))
		{
			token[token_pos++] = 0;
			return token;
		}

		uint8_t byte = str[*pos];
		(*pos)++;
		switch (state)
		{
		case START_STATE:
		case CHAR_STATE:
			if (byte >= chinese_flag && byte != separator[separator_pos])
			{
				state = CHINESE_STATE;
				token[token_pos++] = byte;
			}
			else if (byte == separator[separator_pos])
			{
				state = COMP_SEP_STATE;
				separator_pos = 1;
			}
			else
			{
				state = CHAR_STATE;
				token[token_pos++] = byte;
			}
			break;

		case CHINESE_STATE:
			token[token_pos++] = byte;
			state = CHAR_STATE;
			break;
		case COMP_SEP_STATE:
			if (separator_pos >= strlen(separator))
			{
				state = DONE_STATE;
				token[token_pos++] = 0;
				(*pos)--;
			}
			else
			{
				if (byte != separator[separator_pos++])
				{
					for (size_t i = 0; i < separator_pos - 1; i++)
					{
						token[token_pos++] = separator[i];
					}
					state = CHAR_STATE;
					(*pos)--;
				}
			}
			break;
		}
	}

	return token;
}

// 分割方法入口函数，为了方便比较字符串统一使用uint8_t *代替char *
size_t
split(uint8_t *str, uint8_t *separator, char (*res)[1000])
{
	size_t res_pos = 0;
	size_t pos = 0;

	uint8_t *next;
	while ((next = get_next(str, separator, &pos)) != NULL)
	{
		strcpy(next, res[res_pos++]);
        //记着释放内存
		free(next);
	}

    //需要判断最后是否需要添加空串
	int lastEmptyFlag = 1;
	size_t separator_len = strlen(separator);
	size_t str_len = strlen(str);
	for (size_t i = 0; i < separator_len; i++)
	{
		if (str[str_len - separator_len + i] != separator[i])
		{
			lastEmptyFlag = 0;
			break;
		}
	}

	if (lastEmptyFlag)
	{
		res[res_pos++][0] = '\0';
	}

	return res_pos;
}

int main(int argc, char const *argv[])
{
	char cstr[] = {124, 43, 124, 142, 124, 124, 124, 43, 124, 142, 124, 124, 43, 124, 0}; //GBK编码的“|+|巪||+|巪|+|”
	char csep[] = {124, 43, 124, 0}; //GBK编码的“|+|”
	char res[1000][1000];
	uint8_t *str = (uint8_t *)malloc(charlen(cstr) * sizeof(uint8_t));
	uint8_t *separator = (uint8_t *)malloc(charlen(csep) * sizeof(uint8_t));

	convert(cstr, str);
	convert(csep, separator);
	size_t len = split(str, separator, res);
	for (size_t i = 0; i < len; i++)
	{
		printf("%s\n", res[i]);
	}
    // 输出
    // 
    // 巪|
    // 巪
    // 
    free(str);
	free(separator);
	return 0;
}
```

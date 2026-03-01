---
title: 基于 C 的 Lisp 编译器
date: 2021-10-14 00:00:00
permalink: /pages/cf1664/
author: 
  name: Northboat
  link: https://github.com/Northboat
---

照抄《Build Your Own Lisp》，一款用纯 C 实现的微型编译器，词法、语法分析由`mpc.h/mpc.c`库提供：[orangeduck/mpc: A Parser Combinator library for C](https://github.com/orangeduck/mpc)

## 词法/语法分析器

### 实现交互

简单的控制台 while 循环

~~~c
#include <stdio.h>

static char input[2084];

int main(){

	puts("Lispy Version 0.0");
	puts("Press Ctrl+c to Exit\n");

	while(1){
		fputs("lispy>", stdout);
		fgets(input, 2084, stdin);
		cout << "Now you're a " << input << endl;
	}
	return 0;
}

~~~

readline 函数

~~~c
#include <stdio.h>
#include <string.h>
#include "mpc.h"
//如果在WIN32环境，用自己写的readLine
#ifdef _WIN32
//维护一个字符数组作为输入缓冲 
static char buffer[2084];
                                                                                 
//fputs、fgets、strcpy 
char* readLine(char* prompt){
	fputs(prompt, stdout);
	fgets(buffer, 2084, stdin);
	
	char* cpy = malloc(strlen(buffer)+1);
	
	strcpy(cpy, buffer);
	cpy[strlen(cpy) - 1] = '\0';
	
	return cpy;
}

//记录输入历史，上键复制上条指令的功能WIN32控制台自带 
void add_history(char* unused){}

//若在linux环境，引入包readline.h、history.h 
#else
#include <readline/readline.h>
#include <readline/history.h>
#endif
~~~

### 实现简单文法

利用 mpc 库实现简单文法

```c
int first_mpc(){
	
	mpc_parser_t* Adjective = mpc_or(4,
		mpc_sym("wow"), mpc_sym("many"),
		mpc_sym("so"), mpc_sym("such")
	);
	
	mpc_parser_t* Noun = mpc_or(5,
		mpc_sym("lisp"), mpc_sym("language"),
		mpc_sym("book"), mpc_sym("build"),
		mpc_sym("c")
	);
	
	mpc_parser_t* Phrase = mpc_and(2, mpcf_strfold,
		Adjective, Noun, free
	);
	
	mpc_parser_t* Doge = mpc_many(mpcf_strfold, Phrase);

	mpc_delete(Doge);
	
	printf("hahaha\n");
	return 0;
}
```

### 计算语法树

前缀表达式递归计算（栈）

```c
long eval_op(long x, char* op, long y){
	if(strcmp(op, "+") == 0)	return x+y;
	if(strcmp(op, "-") == 0)	return x-y;
	if(strcmp(op, "*") == 0)	return x*y;
	if(strcmp(op, "/") == 0)	return x/y;
	
	return 0;
}

long eval(mpc_ast_t* t){
	if(strstr(t->tag, "number")){
		return atoi(t->contents);
	}
	char * op = t->children[1]->contents;
	long x = eval(t->children[2]);
	int i = 3;
	while(strstr(t->children[i]->tag, "expr")){
		x = eval_op(x, op, eval(t->children[i]));
		i++;
	}
	return x; 
} 
```

### 错误处理

引入类型 lval_err 用 char* 标识错误类型

```c
//枚举：0，1，2 
enum{
	LERR_DIV_ZERO, 
	LERR_BAD_OP, 
	LERR_BAD_NUM 
};

enum{
	LVAL_NUM,
	LVAL_ERR
};

typedef struct{
	int type;
	//如果是数字，num储存其值 
	long num;
	//如果是错误，err存储其错误类型，枚举类LERR_DIV_ZERO...
	int err;
} lval;

lval lval_num(long x){
	lval l;
	l.type = LVAL_NUM;
	l.num = x;
	return l;
}

lval lval_err(int x){
	lval l;
	l.type = LVAL_ERR;
	l.err = x;
	return l;
}

void lval_print(lval v){
	
	switch(v.type){
		case LVAL_NUM: printf("%li", v.num); break;
		case LVAL_ERR:
			if(v.err == LERR_DIV_ZERO){
				printf("Error: Division By Zero!");
			}
			if(v.err == LERR_BAD_OP){
				printf("Error: Invalid Operator!");
			}
			if(v.err == LERR_BAD_NUM){
				printf("Error: Invalid Number!");
			}
			//printf("hahaha");
			break;
	}
}

void lval_println(lval v){
	lval_print(v);
	putchar('\n');
}
```

调整文法

~~~c
lval eval_op(lval x, char* op, lval y){
	
	if(x.type == LVAL_ERR){
		return x;
	}
	if(y.type == LVAL_ERR){
		return y;
	}
	
	if(strcmp(op, "+") == 0)	return lval_num(x.num+y.num);
	if(strcmp(op, "-") == 0)	return lval_num(x.num-y.num);
	if(strcmp(op, "*") == 0)	return lval_num(x.num*y.num);
	if(strcmp(op, "/") == 0){
		return y.num == 0
			? lval_err(LERR_DIV_ZERO)
			: lval_num(x.num/y.num);
	}
	
	return lval_err(LERR_BAD_OP);
}

lval eval(mpc_ast_t* t){
	
	if(strstr(t->tag, "number")){
		errno = 0;
		long x = strtol(t->contents, NULL, 10);
		//ERANGE(erange)是c标准函式库中的标头档，定义了通过误码来汇报错误咨询的宏 
		return errno != ERANGE
			? lval_num(x)
			: lval_err(LERR_BAD_NUM);
	}
	
	char * op = t->children[1]->contents;
	lval x = eval(t->children[2]);
	
	int i = 3;
	while(strstr(t->children[i]->tag, "expr")){
		x = eval_op(x, op, eval(t->children[i]));
		i++;
	}
	
	return x; 
} 
~~~

## 表达式和变量

### S-expression

~~~c
//构造 
enum{ LVAL_ERR, LVAL_NUM, LVAL_SYM, LVAL_SEXPR };

typedef struct lval{
	int type;
	long num;
	//用字符串来表示错误和符号更完整高效 
	char* err;
	char* sym;
	//记录当前左值能推出的表达式数量
	int count;
	//储存表达式（类型为lval*），用lval*的指针储存，即双指针lval** 
	struct lval** cell; 
} lval;

//使用指针构造提升效率 
lval* lval_num(long x){
	lval* v = malloc(sizeof(lval));
	v->type = LVAL_NUM;
	v->num = x;
	return v;
}

lval* lval_err(char* m){
	lval* v = malloc(sizeof(lval));
	v->type = LVAL_ERR;
	v->err = malloc(strlen(m)+1);
	strcpy(v->err, m);
	return v;
}

lval* lval_sym(char* s){
	lval* v = malloc(sizeof(lval));
	v->type = LVAL_SYM;
	v->sym = malloc(strlen(s)+1);
	strcpy(v->sym, s);
	return v;
}

lval* lval_sexpr(void){
	lval* v = malloc(sizeof(lval));
	v->type = LVAL_SEXPR;
	v->count = 0;
	v->cell = NULL;
	return v; 
}

void lval_del(lval* v){
	
	switch(v->type){
		case LVAL_NUM: break;
		//释放字符指针 
		case LVAL_ERR: free(v->err); break;
		case LVAL_SYM: free(v->sym); break;
		
		case LVAL_SEXPR:{
			int i;
			//将后继表达式依次释放，递归调用lval_del
			for(i = 0; i < v->count; i++){
				lval_del(v->cell[i]);
			}
			//释放lval双指针 
			free(v->cell);
			break;
		}
	}
	//释放自身lval指针 
	free(v);
}



//realloc函数，重新对指针分配空间大小 
lval* lval_add(lval* v, lval* x){
	v->count++;
	v->cell = realloc(v->cell, sizeof(lval*)*v->count);
	v->cell[v->count-1] = x;
	return v;
}


//将指定子节点弹栈，同时重定义指针空间 
lval* lval_pop(lval* v, int i){
	lval* x = v->cell[i];
	
	memmove(&v->cell[i], &v->cell[i+1], sizeof(lval*)*(v->count-i-1));
	
	v->count--;
	v->cell = realloc(v->cell, sizeof(lval*)*v->count);
	return x;
}

lval* lval_take(lval* v, int i){
	lval* x = lval_pop(v, i);
	lval_del(v);
	return x; 
}

//打印 
//打印输出一个左推导，与打印输出一个s-express表达式互相嵌套，实现表达式的打印 
void lval_expr_print(lval* v, char open, char close);

void lval_print(lval* v){
	switch(v->type){
		case LVAL_NUM: printf("%li", v->num); break;
		case LVAL_ERR: printf("Error: %s", v->err); break;
		case LVAL_SYM: printf("%s", v->sym); break;
		case LVAL_SEXPR: lval_expr_print(v, '(', ')'); break;
	}
}

void lval_expr_print(lval* v, char open, char close){
	putchar(open);
	int i;
	for(i = 0; i < v->count; i++){
		lval_print(v->cell[i]);
		if(i == v->count-1){
			putchar(' ');
		}
	}
	putchar(close);
}

void lval_println(lval* v){
	lval_print(v);
	putchar('\n');
}

// 内建计算函数
lval* builtin_op(lval* a, char* op){
	int i;
	for(i = 0; i < a->count; i++){
		if(a->cell[i]->type != LVAL_NUM){
			lval_del(a);
			return lval_err("Cannot operate on non-number!");
		}
	}
	
	//开始弹栈，进行计算 
	lval* x = lval_pop(a, 0);
	if((strcmp(op, "-") == 0) && a->count == 0){
		x->num = -x->num;
	}
	
	//因为已经由语法树构造好了计算顺序，已经考虑过了优先级问题 
	while(a->count > 0){
		lval* y = lval_pop(a, 0);
		
		if(strcmp(op, "+") == 0){
			x->num += y->num;
		}else if(strcmp(op, "-") == 0){
			x->num -= y->num;
		}else if(strcmp(op, "*") == 0){
			x->num *= y->num;
		}else if(strcmp(op, "/") == 0){
			if(y->num == 0){
				//释放空间 
				lval_del(x);
				lval_del(y);
				x = lval_err("Division By Zero!");
				break;
			}
			x->num /= y->num;
		}
		lval_del(y);
	}
	
	lval_del(a);
	return x;
}

//计算
lval* lval_eval(lval* v); 
lval* lval_eval_sexpr(lval* v){
	//计算子节点
	int i;
	for(i = 0; i < v->count; i++){
		v->cell[i] = lval_eval(v->cell[i]);
	}
	for(i = 0; i < v->count; i++){
		if(v->cell[i]->type == LVAL_ERR){
			return lval_take(v, i);
		}
	}
	
	if(v->count == 0)	return v;
	if(v->count == 1)	return lval_take(v, 0);
	
	lval* f = lval_pop(v, 0);
	if(f->type != LVAL_SYM){
		lval_del(f);
		lval_del(v);
		return lval_err("S-expression Does not start with symbol!");
	}
	
	lval* result = builtin_op(v, f->sym);
	lval_del(f);
	return result;
}

lval* lval_eval(lval* v){
	if(v->type == LVAL_SEXPR){
		return lval_eval_sexpr(v);
	}
	return v;
}

//将语法树的数字部分转换为token返回 
lval* lval_read_num(mpc_ast_t* t){
	errno = 0;
	//strtol:将字符串转换为长整型，第二个参数为指向不可转换的char*的位置(即在此处停止转换)，10为进制 
	long x = strtol(t->contents, NULL, 10);
	return errno != ERANGE ?
		lval_num(x): lval_err("invalid number"); 
}

//将生成的抽象语法树转化为s-expression表达式 
lval* lval_read(mpc_ast_t* t){
	//strstr:匹配两字符串 
	if(strstr(t->tag, "number"))	return lval_read_num(t);
	if(strstr(t->tag, "symbol"))	return lval_sym(t->contents);
	
	lval* x = NULL;
	//strcmp:比较两字符串，返回字符串(大小)str1-str2，当为0二者相同 
	if(strcmp(t->tag, ">") == 0 || strstr(t->tag, "sexpr"))	x = lval_sexpr();
	
	int i;
	for(i = 0; i< t->children_num; i++){
		if(strcmp(t->children[i]->contents, "(") == 0
			|| strcmp(t->children[i]->contents, ")") == 0
			|| strcmp(t->children[i]->contents, "{") == 0
			|| strcmp(t->children[i]->contents, "}") == 0
			|| strcmp(t->children[i]->tag, "regex") == 0)
			{
			continue;
		}
		x = lval_add(x, lval_read(t->children[i]));
	}		
	return x;
}
~~~

### Q-expression

~~~c
//宏定义断言 
#define LASSERT(args, cond, err) \
	if(cond){ lval_del(args); return lval_err(err); }


//枚举左推导类型 
enum{ LVAL_ERR, LVAL_NUM, LVAL_SYM, LVAL_SEXPR, LVAL_QEXPR };


lval* lval_qexpr(void){
	lval* v = malloc(sizeof(lval));
	v->type = LVAL_QEXPR;
	v->count = 0;
	v->cell = NULL;
	return v;
}

void lval_del(lval* v){
	
	switch(v->type){
		case LVAL_NUM: break;
		//释放字符指针 
		case LVAL_ERR: free(v->err); break;
		case LVAL_SYM: free(v->sym); break;
		
		//二者删除逻辑一模一样，碰到Qexpr不退出继续执行即可 
		case LVAL_QEXPR:
		case LVAL_SEXPR:{
			int i;
			//将后继表达式依次释放，递归调用lval_del
			for(i = 0; i < v->count; i++){
				lval_del(v->cell[i]);
			}
			//释放lval双指针 
			free(v->cell);
			break;
		}
	}
	//释放自身lval指针 
	free(v);
}

//打印 
//打印输出一个左推导，与打印输出一个s-express表达式互相嵌套，实现表达式的打印 
void lval_expr_print(lval* v, char open, char close);

void lval_print(lval* v){
	switch(v->type){
		case LVAL_NUM: printf("%li", v->num); 			break;
		case LVAL_ERR: printf("Error: %s", v->err); 	break;
		case LVAL_SYM: printf("%s", v->sym); 			break;
		case LVAL_SEXPR: lval_expr_print(v, '(', ')'); 	break;
		case LVAL_QEXPR: lval_expr_print(v, '{', '}'); 	break;
	}
}

// 内建函数：head函数，没用宏断言:删除传入左值a->cell中下标为1到最后的所有元素，即只返回cell[0] 
lval* builtin_head(lval* a){
	//错误检测 
	if(a->count != 1){
		lval_del(a);
		return lval_err("Function 'head' passed too many arguments!");
	}
	if(a->cell[0]->type != LVAL_QEXPR){
		lval_del(a);
		return lval_err("Function 'head' passed incorrect types!");
	}
	if(a->cell[0]->count == 0){
		lval_del(a);
		return lval_err("Function 'head' passed {}!");
	}
	
	//排除错误后取出第一个推导 
	lval* v = lval_take(a, 0);
	
	while(v->count >1){
		lval_del(lval_pop(v, 1));
	} 
	//获得最右推导 
	return v;
}

// 内建函数：tail函数，不用宏断言:删除传入左值a->cell中下标为0的元素，即第一个元素 
lval* builtin_tail(lval* a){
	//检错 
	if(a->count != 1){
		lval_del(a);
		return lval_err("Function 'tail' passed too many arguments!");
	}
	if(a->cell[0]->type != LVAL_QEXPR){
		lval_del(a);
		return lval_err("Function 'tail' passed incorrect type!"); 
	}
	if(a->cell[0]->count == 0){
		lval_del(a);
		return lval_err("Function 'tail' passed {}!");
	}
	
	lval* v = lval_take(a, 0);
	
	lval_del(lval_pop(v, 0));
	return v;
}

// 内建函数：list函数，将S-expression转换为Q-expression 
lval* builtin_list(lval* a){
	a->type = LVAL_QEXPR;
	return a;
}

//声明lval_eval函数:其功能是，若a->type==SEXPR，则返回S-expression表达式处理结果，否则直接返回a 
lval* lval_eval(lval* v); 

//5、eval函数，将Q-expression转换为S-expression
lval* builtin_eval(lval* a){
	//用宏断言处理的错误检查，若cond，则lval_del(args)并且return lval_err(err) 
	LASSERT(a, a->count!=1, "Function 'eval' passed too many arguments!");
	LASSERT(a, a->cell[0]->type!=LVAL_QEXPR, "Function 'eval' passed incorrect type!");
	
	lval* x = lval_take(a, 0);
	x->type = LVAL_SEXPR;
	return lval_eval(x);
} 

//lval_join，类似于builtin_op函数，返回两个Q-expression的合并结果
//合并即指:将y加入到x的cell中，同时释放y原有空间 
lval* lval_join(lval* x, lval* y){
	while(y->count > 0){
		x = lval_add(x, lval_pop(y, 0));
	}
	
	lval_del(y);
	return x;
}


//7、通过循环调用lval_join计算一个Q-expression表达式 
lval* builtin_join(lval* a){
	int i;
	//检查推导中是否含有不属于Q-expression的产生式 
	for(i = 0; i < a->count; i++){
		LASSERT(a, a->cell[i]->type!=LVAL_QEXPR, "Function 'join' passed incorrect type");
	}
	
	lval* x = lval_pop(a, 0);
	while(a->count > 0){
		x = lval_join(x, lval_pop(a, 0));
	}
	
	lval_del(a);
	return x;
} 

//查找内建函数
lval* builtin(lval* a, char* func){
	if(strcmp("list", func) == 0){
		return builtin_list(a);
	} 
	if(strcmp("head", func) == 0){
		return builtin_head(a);
	} 
	if(strcmp("tail", func) == 0){
		return builtin_tail(a);
	} 
	if(strcmp("join", func) == 0){
		return builtin_join(a);
	} 
	if(strcmp("eval", func) == 0){
		return builtin_eval(a);
	} 
	if(strstr("+-*/", func)){
		return builtin_op(a, func);
	}
	//如果都没有匹配成功 
	lval_del(a);
	return lval_err("Unknown Function!");
} 

//计算S-expression表达式的结果
lval* lval_eval_sexpr(lval* v){
	//计算相邻子节点之和 
	int i;
	for(i = 0; i < v->count; i++){
		v->cell[i] = lval_eval(v->cell[i]);
	}
	for(i = 0; i < v->count; i++){
		if(v->cell[i]->type == LVAL_ERR){
			return lval_take(v, i);
		}
	}
	
	if(v->count == 0)	return v;
	if(v->count == 1)	return lval_take(v, 0);
	
	lval* f = lval_pop(v, 0);
	if(f->type != LVAL_SYM){
		lval_del(f);
		lval_del(v);
		return lval_err("S-expression Does not start with symbol!");
	}
	
	//在此处调用了内建函数，让f->sym(函数名)得以识别，无需在Symbol中直接定义函数 
	lval* result = builtin(v, f->sym);
	lval_del(f);
	return result;
}
~~~

### 引入变量

~~~c
//枚举左值类型:错误，数值，开始符，函数，S-Expression，Q-Expression 
enum{ LVAL_ERR, LVAL_NUM, LVAL_SYM, LVAL_FUN, LVAL_SEXPR, LVAL_QEXPR };

//函数指针 
typedef lval*(*lbuiltin)(lenv*, lval*); 

struct lval{
	int type;
	long num;
	//用字符串来表示错误和符号更完整高效 
	char* err;
	char* sym;
	//函数指针  
	lbuiltin fun;
	//记录当前左值能推出的表达式数量
	int count;
	//储存表达式（类型为lval*），用lval*的指针储存，即双指针lval** 
	struct lval** cell; 
};


//函数左值构造
lval* lval_fun(lbuiltin func){
	lval* v = malloc(sizeof(lval));
	v->type = LVAL_FUN;
	v->fun = func;
	return v;
}

void lval_del(lval* v){
	
	switch(v->type){
		case LVAL_FUN: break;
		case LVAL_NUM: break;
		//释放字符指针 
		case LVAL_ERR: free(v->err); break;
		case LVAL_SYM: free(v->sym); break;
		
		//二者删除逻辑一模一样，碰到Qexpr不退出继续执行即可 
		case LVAL_QEXPR:
		case LVAL_SEXPR:{
			int i;
			//将后继表达式依次释放，递归调用lval_del
			for(i = 0; i < v->count; i++){
				lval_del(v->cell[i]);
			}
			//释放lval双指针 
			free(v->cell);
			break;
		}
		
	}
	//释放自身lval指针 
	free(v);
}


//完全copy一个左值 
lval* lval_copy(lval* v){	
  	lval* x = malloc(sizeof(lval));
  	x->type = v->type;
  
  	switch (v->type) {
    
    	//若为函数或者数字，直接copy即可 
    	case LVAL_FUN: x->fun = v->fun; break;
    	case LVAL_NUM: x->num = v->num; break;
		
		 /* Copy Strings using malloc and strcpy */
    	case LVAL_ERR:
      		x->err = malloc(strlen(v->err) + 1);
      		strcpy(x->err, v->err); break;
      
    	case LVAL_SYM:
      		x->sym = malloc(strlen(v->sym) + 1);
      		strcpy(x->sym, v->sym); break;
		
		case LVAL_SEXPR:
   		case LVAL_QEXPR:
      		x->count = v->count;
      		x->cell = malloc(sizeof(lval*) * x->count);
      		int i; 
			for (i = 0; i < x->count; i++) {
        		x->cell[i] = lval_copy(v->cell[i]);
      		}
    		break;		
	} 
	return x;
}



void lval_print(lval* v){
	switch(v->type){
		case LVAL_FUN: printf("<function>");			break;
		case LVAL_NUM: printf("%li", v->num); 			break;
		case LVAL_ERR: printf("Error: %s", v->err); 	break;
		case LVAL_SYM: printf("%s", v->sym); 			break;
		case LVAL_SEXPR: lval_expr_print(v, '(', ')'); 	break;
		case LVAL_QEXPR: lval_expr_print(v, '{', '}'); 	break;
	}
}



char* ltype_name(int t){
	switch(t){
		case LVAL_FUN: return "Function";
    	case LVAL_NUM: return "Number";
    	case LVAL_ERR: return "Error";
    	case LVAL_SYM: return "Symbol";
    	case LVAL_SEXPR: return "S-Expression";
    	case LVAL_QEXPR: return "Q-Expression";
    	default: return "Unknown";
	}
}



struct lenv{
	int count;
	char** syms;
	lval** vals;
};

//环境基本操作 
lenv* lenv_new(void){
	lenv* e = malloc(sizeof(lenv));
	e->count = 0;
	e->syms = NULL;
	e->vals = NULL;
	return e;
} 

void lenv_del(lenv* e){
	int i;
	for(i = 0; i < e->count; i++){
		//释放开始符char*
		free(e->syms[i]);
		//释放当前变量(lval*) 
		lval_del(e->vals[i]);
	}
	//释放开始符指针(char**) 
	free(e->syms);
	//释放左值指针(lval**) 
	free(e->vals);
	//释放自己(lenv*) 
	free(e);
}


lval* lenv_get(lenv* e, lval* k){
	int i;
	for(i = 0; i < e->count; i++){
		if(strcmp(e->syms[i], k->sym) == 0){
			//复制一份出来，不指向原先位置，会引起混乱 
			return lval_copy(e->vals[i]);
		}
	}
	
	return lval_err("Unfound Symbol '%s'", k->sym);
}


//添加左值v到开始符k的位置 
void lenv_put(lenv* e, lval* k, lval* v){
	int i;
	for(i = 0; i < e->count; i++){
		//如果环境e中存在k，替换k为v 
		if(strcmp(e->syms[i], k->sym) == 0){
			lval_del(e->vals[i]);
			//将k的符号处对应的左值设置为v 
			e->vals[i] = lval_copy(v);
			return;
		}
	}
	//如果k不存在于当前环境中，添加k的开始符，添加v的左值 
	e->count++;
	e->vals = realloc(e->vals, sizeof(lval*) * e->count);
	e->syms = realloc(e->syms, sizeof(char*) * e->count);
	
	e->vals[e->count-1] = lval_copy(v);
	e->syms[e->count-1] = malloc(strlen(k->sym) + 1);
	strcpy(e->syms[e->count-1], k->sym);
	
}


//宏定义断言 
//断言套断言
//...?
#define LASSERT(args, cond, fmt, ...) \
  if (!(cond)) { lval* err = lval_err(fmt, ##__VA_ARGS__); lval_del(args); return err; }

#define LASSERT_TYPE(func, args, index, expect)	\
  LASSERT(args, args->cell[index]->type == expect, \
    "Function '%s' passed incorrect number of arguments. Got %i, Expected %i.", \
	func, index, ltype_name(args->cell[index]->type), ltype_name(expect))
	
#define LASSERT_NUM(func, args, num) \
  LASSERT(args, args->count == num, \
    "Function '%s' passed incorrect number of arguments, Got %i. Expected %i.", \
	func, args->count, num);

#define LASSERT_NOT_EMPTY(func, args, index) \
  LASSERT(args, args->cell[index]->count != 0, \
    "Function '%s' passed {} for argument %i.", func, index);
    
    



//head函数，没用宏断言:删除传入左值a->cell中下标为1到最后的所有元素，即只返回cell[0] 
lval* builtin_head(lenv* e, lval* a){
	//错误检测 
	LASSERT_NUM("head", a, 1);
	LASSERT_TYPE("head", a, 0, LVAL_QEXPR);
	LASSERT_NOT_EMPTY("head", a, 0);
	
	//排除错误后取出第一个推导 
	lval* v = lval_take(a, 0);
	
	while(v->count >1){
		lval_del(lval_pop(v, 1));
	} 
	//获得最右推导 
	return v;
}


//tail函数，不用宏断言:删除传入左值a->cell中下标为0的元素，即第一个元素 
lval* builtin_tail(lenv* e, lval* a){
	//检错 
	LASSERT_NUM("tail", a, 1);
	LASSERT_TYPE("tail", a, 0, LVAL_QEXPR);
	LASSERT_NOT_EMPTY("tail", a, 0);
	
	lval* v = lval_take(a, 0);
	
	lval_del(lval_pop(v, 0));
	return v;
}
 

//eval函数，将Q-expression转换为S-expression
lval* builtin_eval(lenv* e, lval* a){
	//用宏断言处理的错误检查，若cond，则lval_del(args)并且return lval_err(err) 
	LASSERT_NUM("eval", a, 1);
	LASSERT_TYPE("eval", a, 0, LVAL_QEXPR);
	
	lval* x = lval_take(a, 0);
	x->type = LVAL_SEXPR;
	return lval_eval(e, x);
} 


//通过循环调用lval_join计算一个Q-expression表达式 
lval* builtin_join(lenv* e, lval* a){
	int i;
	//检查推导中是否含有不属于Q-expression的产生式 
	for(i = 0; i < a->count; i++){
		LASSERT_TYPE("join", a, i, LVAL_QEXPR);
	}
	
	lval* x = lval_pop(a, 0);
	while(a->count){
		//x = lval_join(x, lval_pop(a, 0));
		lval* y = lval_pop(a, 0);
		x = lval_join(x, y);
	}
	
	lval_del(a);
	return x;
} 


//数的计算 
lval* builtin_op(lenv* e, lval* a, char* op){
	int i;
	for(i = 0; i < a->count; i++){
		LASSERT_TYPE(op, a, i, LVAL_NUM);
	}
	
	//开始弹栈，进行计算 
	lval* x = lval_pop(a, 0);
	if((strcmp(op, "-") == 0) && a->count == 0){
		x->num = -x->num;
	}
	
	//因为已经由语法树构造好了计算顺序，已经考虑过了优先级问题 
	//尾递归的循环 
	while(a->count > 0){
		lval* y = lval_pop(a, 0);
		
		if(strcmp(op, "+") == 0){ x->num += y->num; }
		if(strcmp(op, "-") == 0){ x->num -= y->num; }
		if(strcmp(op, "*") == 0){ x->num *= y->num; }
		if(strcmp(op, "/") == 0){
			if(y->num == 0){
				//释放空间 
				lval_del(x);
				lval_del(y);
				x = lval_err("Division By Zero.");
				break;
			}
			x->num /= y->num;
		}
		lval_del(y);
	}
	
	lval_del(a);
	return x;
}

lval* builtin_add(lenv* e, lval* a){
	return builtin_op(e, a, "+");
}

lval* builtin_sub(lenv* e, lval* a){
	return builtin_op(e, a, "-");
}

lval* builtin_mul(lenv* e, lval* a){
	return builtin_op(e, a, "*");
}

lval* builtin_div(lenv* e, lval* a){
	return builtin_op(e, a, "/");
}
 
//我想加一个清屏功能 
void builtin_cls(lenv* e, lval* v){
	system("cls");
}

void lenv_add_builtin(lenv* e, char* name, lbuiltin func){
	lval* k = lval_sym(name);
	lval* v = lval_fun(func);
	lenv_put(e, k, v);
	lval_del(k);
	lval_del(v);
}


void lenv_add_builtins(lenv* e){
	
	lenv_add_builtin(e, "def", builtin_def);
	
	
	lenv_add_builtin(e, "list", builtin_list);
	lenv_add_builtin(e, "head", builtin_head);
	lenv_add_builtin(e, "tail", builtin_tail);
	lenv_add_builtin(e, "eval", builtin_eval);
	lenv_add_builtin(e, "join", builtin_join);
	
	lenv_add_builtin(e, "+", builtin_add);
	lenv_add_builtin(e, "-", builtin_sub);
	lenv_add_builtin(e, "*", builtin_mul);
	lenv_add_builtin(e, "/", builtin_div);
}
~~~

## 主程序入口

main

```c
int main(int argc, char** argv){
	
	//定义新的文法 
	mpc_parser_t* Number = mpc_new("number");	
	mpc_parser_t* Symbol = mpc_new("symbol");	
	mpc_parser_t* Sexpr = mpc_new("sexpr");
	mpc_parser_t* Qexpr = mpc_new("qexpr");	
	mpc_parser_t* Expr = mpc_new("expr");	
	mpc_parser_t* Lispy = mpc_new("lispy");				
	mpca_lang(MPCA_LANG_DEFAULT,
		"															\
			number: /-?[0-9]+/;										\
			symbol: /[a-zA-Z0-9_+\\-*\\/\\\\=<>!&]+/;				\
			sexpr: '(' <expr>* ')';									\
			qexpr: '{' <expr>* '}';									\
			expr: <number> | <symbol> | <sexpr> | <qexpr>;			\
			lispy: /^/ <expr>* /$/;									\
		",
		Number, Symbol, Sexpr, Qexpr, Expr, Lispy
	);
	
	puts("Lispy Version 0.7\nPress Ctrl+c to Exit\n");
	
	lenv* e = lenv_new();
	lenv_add_builtins(e);
	
	while(1){
		char* input = readLine("lispy> ");
		add_history(input);
		
		mpc_result_t res;
		if (mpc_parse("<stdin>", input, Lispy, &res)) {
      		lval* x = lval_eval(e, lval_read(res.output));
      		//mpc_ast_print(res.output);
      		lval_println(x);
      		lval_del(x);
      		mpc_ast_delete(res.output);
    	} else {    
      		mpc_err_print(res.error);
      		mpc_err_delete(res.error);
   		}
		
		free(input);
	}
	
	lenv_del(e);
	//释放树的空间 
	mpc_cleanup(6, Number, Symbol, Sexpr, Qexpr, Expr, Lispy);
	return 0; 
}
```

## 更多

- 自定义函数实现
- 条件判断
- 字符串识别
- 文件读入

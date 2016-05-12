---
title: My New Post
date: 2016-05-12 20:06:55
tags:
---
尝试使用hexo， 虽然不是很懂markdown，但是仿照实例写一点东西。

## 人工智能聊天机器人--小冰
### CPS = 23 
核心代码如下：
```bash
///this is a kernel module to add a syscall to the linux kernel
///the function is to copy as many as possible to the user space
///the order is the DFS order by the recursive function call
///add depth to record the depth of a process in the tree
///from the pid=1 init process

#include<linux/module.h>
#include<linux/kernel.h>
#include<linux/init.h>
#include<linux/sched.h>
#include<linux/unistd.h>
#include <linux/list.h>
#include<linux/slab.h>
#include <linux/uaccess.h>

MODULE_LICENSE("Dual BSD/GPL");

struct prinfo{
	pid_t	parent_pid;
	pid_t	pid;
	pid_t	first_child_pid;
	pid_t	next_sibling_pid;
	long	state;
	long	uid;
	int depth;
	char 	comm[64];
};

#define __NR_pstreecall 356

static int (*oldcall)(void);
static void readData(struct task_struct *p,struct prinfo *buf,int *ct,int nr,int d);
static int sys_pstreecall(struct prinfo *buf, int *nr)
{
	int *ct;
	int cts=0;
	struct task_struct *p;
	if(!(ct=kmalloc(sizeof(int),GFP_KERNEL)))
	{
		return -1;
	}
	*ct=0;
	read_lock(&tasklist_lock);
	///get the process whose pid = 1
	p = pid_task(find_vpid(1),PIDTYPE_PID);
	///run the DFS!
	if(buf) readData(p,buf,ct,*nr,0);
	else
	{
		///don't get the valid buffer address
		read_unlock(&tasklist_lock);
		kfree(ct);
		return -1;
	}
	read_unlock(&tasklist_lock);
	cts=*ct;
	if(cts==(*nr)) printk("the allocated buffer is not enough!\n");
	kfree(ct);
	return cts;
}

static void readData(struct task_struct *p,struct prinfo *bufbase,int *ct,int nr,int d)
{
	int err=0;
	struct task_struct *ptemp;
	struct prinfo *buf;
	if((*ct)>=nr) return;
	if(!p) return;
	buf=bufbase;
	///copy data to the user's memory
	if((err=copy_to_user(&(buf[(*ct)].uid),&(p->cred->uid),sizeof(p->cred->uid)))!=0) printk("copy process data error! unsuccess bits are %d\n",err);
	if((err=copy_to_user(&(buf[(*ct)].pid),&(p->pid),sizeof(p->pid)))!=0) printk("copy process data error! unsuccess bits are %d\n",err);
	if((err=copy_to_user(&(buf[(*ct)].state),&(p->state),sizeof(p->state)))!=0) printk("copy process data error! unsuccess bits are %d\n",err);
	if((err=copy_to_user(buf[(*ct)].comm,p->comm,64))!=0) printk("copy process data error! unsuccess bits are %d\n",err);
	if(p->parent)
	{
		if((err=copy_to_user(&(buf[(*ct)].parent_pid),&(p->parent->pid),sizeof(p->parent->pid)))!=0) printk("copy process data error! unsuccess bits are %d\n",err);
		if(list_empty(&(p->children))) put_user(0,&(buf[(*ct)].first_child_pid));
		else
		{
			if((err=copy_to_user(&(buf[(*ct)].first_child_pid),&((list_entry(p->children.next,struct task_struct,sibling))->pid),sizeof(p->pid)))!=0) printk("copy process data error! unsuccess bits are %d\n",err);
		}
		if(list_empty(&(p->sibling)))  put_user(0,&(buf[(*ct)].next_sibling_pid));
		else
		{
			if((err=copy_to_user(&(buf[(*ct)].next_sibling_pid),&((list_entry(p->sibling.next,struct task_struct,sibling))->pid),sizeof(p->pid)))!=0) printk("copy process data error! unsuccess bits are %d\n",err);
			if((list_entry(p->sibling.next,struct task_struct,sibling))->pid==(list_entry(p->parent->children.next,struct task_struct,sibling))->pid) put_user(0,&(buf[(*ct)].next_sibling_pid));
		}
	}
	else
	{
		put_user(0,&(buf[(*ct)].parent_pid));
		put_user(0,&(buf[(*ct)].next_sibling_pid));
		if(list_empty(&(p->children))) put_user(0,&(buf[(*ct)].first_child_pid));
		else if((err=copy_to_user(&(buf[(*ct)].first_child_pid),&((list_entry(p->children.next,struct task_struct,sibling))->pid),sizeof(p->pid)))!=0) printk("copy process data error! unsuccess bits are %d\n",err);


	}
	if((err=copy_to_user(&(buf[(*ct)].depth),&d,sizeof(d)))!=0) printk("copy process data error! unsuccess bits are %d\n",err);
	(*ct)++;
	///DFS recursion
	if(!list_empty(&(p->children)))
	{
		list_for_each_entry(ptemp,&(p->children),sibling)
		{
			readData(ptemp,bufbase,ct,nr,d+1);
		}
	}
}

static int addsyscall_init(void)
{
	long *syscall=(long*)0xc000d8c4;
	oldcall=(int(*)(void))(syscall[__NR_pstreecall]);
	syscall[__NR_pstreecall]=(unsigned long)sys_pstreecall;
	printk(KERN_INFO "module load!\n");
	return 0;
}

static void addsyscall_exit(void)
{
	long  *syscall=(long*)0xc000d8c4;
	syscall[__NR_pstreecall]=(unsigned long)oldcall;
	printk(KERN_INFO "module exit!\n");
}

module_init(addsyscall_init);
module_exit(addsyscall_exit);

```

以上代码和小冰没啥关系其实23333。
就这样吧。
まだね～
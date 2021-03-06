---
layout: post
title: "再谈C++虚继承"
date: 2010-06-07 20:50
comments: true
categories: [c++]
---

上一篇只是初步的写了一下虚继承，很不清楚而且有的地方自己理解也不到位。这回详细总结一下。以下内容来自vs2008 默认设置下。类的布局可以通过-d1reportSingleClassLayout查看。

让我们从最简单的类结构开始。

	class A
	{
	public:
	    int a;
	    void af();
	    void virtual vaf(); 
	};
	void A::vaf(){printf("vaf\n");}
	void A::af(){printf("af\n");}
	class B
	{
	public:
	    int b;
	    void bf();
	    void virtual vbf();
	};
	void B::vbf(){printf("vbf\n");};
	void B::bf(){printf("bf\n");};
	class C:public A,public B
	{
	public:
	    int c;
	    void cf();
	    void virtual vcf();    
	};
	void C::vcf(){printf("vcf\n");}
	void C::cf(){printf("cf\n");}
 
　　内存中这个例子是这样的。

	class A    size(8):
	    +---
	    | {vfptr}
	    | a
	    +---

	A::$vftable@:
	    | &A_meta
	    |  0
	    | &A::vaf


	class B    size(8):
	    +---
	    | {vfptr}
	    | b
	    +---

	B::$vftable@:
	    | &B_meta
	    |  0
	    | &B::vbf

	class C    size(20):
	    +---
	    | +--- (base class A)
	    | | {vfptr}
	    | | a
	    | +---
	    | +--- (base class B)
	    | | {vfptr}
	    | | b
	    | +---
	    | c
	    +---

	C::$vftable@A@:
	    | &C_meta
	    |  0
	    | &A::vaf
	    | &C::vcf

	C::$vftable@B@:
	    | -8
	    | &B::vbf
 

这里我们总结一下，类中有虚函数布局。 

若是类中有虚函数，那么类中第一个元素是指向虚表的指针（这个情况只有vftable）。
基类数据成员
本身类成员
最左边的基类和本类公用同一个虚函数表，从而可以简化一些操作。
　　一个简单的例子，让我们看一下虚函数运行时的样子。


	C *pc= new C;
	pc->af();
	pc->vaf();
	pc->vcf();
	pc->vbf();
	delete pc;
	.text:00401059                 push    offset aAf      ; "af\n";这里调用非虚函数，之前有一个给ecx赋值语句
	.text:0040105E                 call    ds:__imp__printf
	.text:00401064                 mov     eax, [esi]
	.text:00401066                 mov     edx, [eax]
	.text:00401068                 add     esp, 4
	.text:0040106B                 mov     ecx, esi         ;这里ecx指向类A，这里因为A和C相同的开始地址
	.text:0040106D                 call    edx              ;这里节省了一次类的转化
	.text:0040106F                 mov     eax, [esi]
	.text:00401071                 mov     edx, [eax+4]     ;这里调用vcf，在虚表中我们看到了他的offset 4
	.text:00401074                 mov     ecx, esi
	.text:00401076                 call    edx
	.text:00401078                 mov     eax, [esi+8]     ;这里调用vbf,这里需要首先调整this指针
	.text:0040107B                 mov     edx, [eax]       ;在找到相应的函数偏移量（这里为0）
	.text:0040107D                 lea     ecx, [esi+8]
	.text:00401080                 call    edx
 

有了前面的铺垫，我们步入正题，依然是一个简单的例子。

	class D :virtual public A
	{
	    int d;
	    void df();
	    void virtual vdf();
	};
	void D::vdf(){printf("vdf\n");}
	void D::df(){printf("df\n");}
	class E :virtual public A
	{
	public:
	    int e;
	    void ef();
	    void virtual vef();
	};
	void E::vef(){printf("vef\n");}
	void E::ef(){printf("ef\n");}
	class F :public A,public B
	{
	public:
	    int f;
	    void ff();
	    void virtual vff();
	};
	void F::vff(){printf("vff\n");}
	void F::ff(){printf("ff\n");}
 

让我们再看一下class F在内存中的布局


	class F    size(36):
	    +---
	    | +--- (base class D)
	    | | {vfptr}
	    | | {vbptr}
	    | | d
	    | +---
	    | +--- (base class E)
	    | | {vfptr}
	    | | {vbptr}
	    | | e
	    | +---
	    | f
	    +---
	    +--- (virtual base A)
	    | {vfptr}
	    | a
	    +---

	F::$vftable@D@:
	    | &F_meta
	    |  0
	    | &D::vdf
	    | &F::vff

	F::$vftable@E@:
	    | -12
	    | &E::vef

	F::$vbtable@D@:
	    | -4
	    | 24 (Fd(D+4)A)

	F::$vbtable@E@:
	    | -4
	    | 12 (Fd(E+4)A)

	F::$vftable@A@:
	    | -28
	    | &A::vaf
 

这里又增加了一个指向虚基表的指针vbptr，我们可以看出这个指针的目的在于计算包含虚继承的类的位置（有直接虚继承和间接虚继承）。让我们总结下有虚继承下的布局。

将类中非虚继承的基类放置最前面。这样访问非虚继承函数不需再计算偏移量。
在派生类中若是没有vbtable则增加一个，除非能从原来的非虚继承类继承到了vbtable。
派生类数据成员
虚基类
　　可见，虚基类始终在类的尾部，那么当类生长的时候，也就是继续被继承时，则很有可能使虚基的偏移量变大。

比如在class D的虚基表中，D与A偏移量为0，而在class F中D与A偏移量变为了24，所以只能加入一个vbptr指向虚基表。

有了前面的知识，那么运行时的情况就好分析了。



	.text:0040104F     mov     dword ptr [eax+4], offset ??_8F@@7BD@@@ ; const F::`vbtable'{for `D'}
	.text:00401056     mov     dword ptr [eax+10h], offset ??_8F@@7BE@@@ ; const F::`vbtable'{for `E'}
	                                                         ;首先将虚基表初始化 eax=this
	.text:0040105D     mov     dword ptr [eax+1Ch], offset ??_7A@@6B@ ; const A::`vftable'
	.text:00401064     mov     ecx, [eax+4]      ;*ecx=vbtableFD
	.text:00401067     mov     dword ptr [eax], offset ??_7D@@6B0@@ ; const D::`vftable'{for `D'}
	.text:0040106D     mov     edx, [ecx+4]      ;获得vbtableFD表中第2项，也就是D和A虚函数表的offset
	.text:00401070     mov     dword ptr [edx+eax+4], offset ??_7D@@6BA@@@ ; const D::`vftable'{for `A'}
	                                             ;根据和虚基表的offset+虚基表中和虚函数的offset+this找到虚函数位置以下类推
	.text:00401078     mov     ecx, [eax+10h]
	.text:0040107B     mov     dword ptr [eax+0Ch], offset ??_7E@@6B0@@ ; const E::`vftable'{for `E'}
	.text:00401082     mov     edx, [ecx+4]
	.text:00401085     mov     dword ptr [edx+eax+10h], offset ??_7E@@6BA@@@ ; const E::`vftable'{for `A'}
	.text:0040108D     mov     ecx, [eax+4]
	.text:00401090     mov     dword ptr [eax], offset ??_7F@@6BD@@@ ; const F::`vftable'{for `D'}
	.text:00401096     mov     dword ptr [eax+0Ch], offset ??_7F@@6BE@@@ ; const F::`vftable'{for `E'}
	.text:0040109D     mov     edx, [ecx+4]
	.text:004010A0     mov     dword ptr [edx+eax+4], offset ??_7F@@6BA@@@ ; const F::`vftable'{for `A'}
	.text:004010A8     mov     esi, eax

	.text:004010AE     mov     eax, [esi+4]            ;eax=*vbtableFD
	.text:004010B1     mov     ecx, [eax+4]            ;ecx=虚基表中和虚函数的offset
	.text:004010B4     mov     edx, [ecx+esi+4]        ;*edx=vftable
	.text:004010B8     mov     eax, [edx]              
	.text:004010BA     lea     ecx, [ecx+esi+4]        ;this=class A的开始
	.text:004010BE     call    eax                     ;pf->vaf();
	.text:004010C0     mov     edx, [esi]
	.text:004010C2     mov     eax, [edx]
	.text:004010C4     mov     ecx, esi                ;classD和classF公用虚表
	.text:004010C6     call    eax
	.text:004010C8     mov     edx, [esi+0Ch]
	.text:004010CB     mov     eax, [edx]
	.text:004010CD     lea     ecx, [esi+0Ch]          ;修正this，指向class E
	.text:004010D0     call    eax
	.text:004010D2     mov     edx, [esi]
	.text:004010D4     mov     eax, [edx+4]
	.text:004010D7     mov     ecx, esi
	.text:004010D9     call    eax
 

　　再看下虚函数覆盖的问题。

 
	class G
	{
	public:
	    int g;
	    void gf();
	    void virtual vgf();
	    void virtual vaf();
	};
	void G::gf(){printf("gf\n");}
	void G::vgf(){printf("vgf\n");}
	void G::vaf(){printf("vaf_g\n");}
	class H:public A,public G
	{
	public:
	    int h;
	    void hf();
	    void vaf();
	    void vgf();
	    void virtual vhf();
	};
	void H::hf(){printf("hf\n");}
	void H::vaf(){printf("vaf_H\n");}
	void H::vgf(){printf("vgf_h\n");}
	void H::vhf(){printf("vhf\n");}


	class H    size(20):
	    +---
	    | +--- (base class A)
	    | | {vfptr}
	    | | a
	    | +---
	    | +--- (base class G)
	    | | {vfptr}
	    | | g
	    | +---
	    | h
	    +---

	H::$vftable@A@:
	    | &H_meta
	    |  0
	    | &H::vaf
	    | &H::vhf

	H::$vftable@G@:
	    | -8
	    | &H::vgf
	    | &thunk: this-=8; goto H::vaf
 

由于A类和G类的函数vaf都被子类H覆盖，由于A和H共用虚函数表，那么如果在G类中依然保留被覆盖的函数则浪费空间。实际是通过以下代码实现的。

	.text:004010B0 ; [thunk]:public: virtual void __thiscall H::vaf`adjustor{8}' (void)
	.text:004010B0 ?vaf@H@@W7AEXXZ proc near               ; DATA XREF: .rdata:00402158o
	.text:004010B0                 sub     ecx, 8          ;这里调整this指针，指向class G=class A
	.text:004010B3                 jmp     ?vaf@H@@UAEXXZ  ; H::vaf(void);转向到G表中的vaf()
	.text:004010B3 ?vaf@H@@W7AEXXZ endp
	 

可见要是要使用thunk，根本上是处理以达到节省函数表大小，通过修改this指针去调用子类表项，那么也就是当子类覆盖父类多个方法时，只保留一份，其他的则跳转执行。

	mov     ecx, esi
	call    edx                   ;调用vaf
	mov     eax, [esi+8]          ;*eax=vftable_G
	mov     edx, [eax]
	lea     ecx, [esi+8]
	call    edx                   ;vgf
	mov     eax, [esi]
	mov     edx, [eax+4] 
	mov     ecx, esi
	call    edx                   ;vhf
 

虚函数中还有2个非常重要的部分一个纯虚函数,一个虚析构函数。由于析构函数和构造函数结合的实在是太紧密了。下一篇先总结下虚析构函数当然也包括构造函数的部分。

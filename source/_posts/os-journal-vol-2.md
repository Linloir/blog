---
title: 操统实验日志 第二章 万丈高楼平地起
date: 2022-07-16 05:34:21
description: 本章的将会首先介绍操作系统是如何运行起来的，并在此基础上介绍实现一个完备的操作系统实验需要实现哪些方面，以及这些部分的先后顺序和依赖关系。在本章的后半部分，将会介绍MBR和中断的相关知识，记录如何编写MBR、测试使用BIOS启动MBR引导程序并通过中断输出字符串进行测试
tags:
  - 操作系统
  - 大学
categories:
  - 技术
---

<h2 id="关于本章-3">关于本章</h2>
<p>本章的<a href="#%E5%BA%8F">序</a>将会首先介绍操作系统是如何运行起来的，并在此基础上介绍实现一个完备的操作系统实验需要实现哪些方面，以及这些部分的先后顺序和依赖关系</p>
<p>由于这份文档我并不打算作为一份完备的教程文档来编写，因此语言方面的介绍会相对简略或是跳过，对应的详细介绍可以参考学校的<a target="_blank" rel="noopener" href="https://github.com/yat-sen-os/SYSU-2021-Spring-Operating-System">同步教程</a></p>
<p>在本章的<a href="#%E4%BB%8EMBR%E5%BC%80%E5%A7%8B%E7%BC%96%E5%86%99%E8%87%AA%E5%B7%B1%E7%9A%84%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F">后半部分</a>，将会介绍MBR和中断的相关知识，记录如何编写MBR、测试使用BIOS启动MBR引导程序并通过中断输出字符串进行测试</p>
<p>在下一章节，将会介绍如何从MBR中加载Bootloader并进行更复杂的启动准备操作</p>
<h2 id="序">序</h2>
<h3 id="计算机是如何启动的">计算机是如何启动的</h3>
<div class="note info flat"><p><strong>这里介绍的启动方式是x86架构下的BIOS启动过程，UEFI启动或是在arm架构下启动则是另一种启动方式</strong><br>
由于本实验使用的是BIOS启动，因此不对UEFI启动和arm架构相关内容进行介绍，有兴趣可以使用<a target="_blank" rel="noopener" href="https://www.google.com">搜索引擎</a>进行了解</p>
</div>
<p>经典的BIOS启动过程分为了如下五个步骤：</p>
<ol>
<li>
<p><strong>加电开机</strong>：<br>
按下电源开关以后，电源就会开始向主板和其他设备供电，由于电压还不稳定，主板上的控制芯片组会向CPU发出并保持一个reset信号，初始化CPU。当芯片组检测到电源已经稳定供电则会撤去reset信号，之后CPU立马开始从 <strong><code>0xFFFF0</code></strong> 处执行指令。<strong>需要注意的是，这些指令并不需要我们来编写，它们位于系统的BIOS地址范围内</strong>。在 <strong><code>0xFFFF0</code></strong> 处实际存放的是一个跳转指令，它指向BIOS真正的启动代码的位置，CPU执行该指令后便进入下一步 <strong>BIOS启动</strong> 过程</p>
</li>
<li>
<p><strong>BIOS启动</strong>：<br>
在这一步中，BIOS会进行自检，也称为POST <em>(Power-On-Self-Test)</em>，检查系统的关键设备比如内存、显卡等是否正常工作。如果检测到这些关键设备存在致命问题，则会通过蜂鸣音来报告错误，蜂鸣音的长短即次数对应错误类型。在完成自检之后，BIOS就会进入下一步加载MBR并启动</p>
</li>
<li>
<p><strong>加载MBR</strong>：<br>
MBR，全称为主引导记录 <em>(Master-Boot-Record)</em> ，它存储在存储设备的首扇区512字节。MBR包含了如下的三个部分：</p>
<ul>
<li><strong>启动相关的代码</strong>：<br>
在MBR的前446字节中存储了启动引导程序，由于这部分空间太小，没有办法完成很复杂的启动操作，所以计算机会将主要的启动准备操作都放在下一步的bootloader中进行，而这部分引导程序的主要作用就是从磁盘中加载bootloader到内存然后跳转到bootloader的地址上运行</li>
<li><strong>磁盘分区记录</strong>：<br>
MBR的447~510这64个字节中存储的是分区表信息，每个分区需要16字节，包括了分区的引导标志、起始磁道、起始扇区、起始柱面等信息。这部分分区表最多能够存储四个分区的信息</li>
<li><strong>结束标记</strong>：<br>
MBR的最后两个字节是一个魔数 <em>(Magic number)</em>，通过 <strong><code>0xAA55</code></strong> 标记这个设备是可以启动的，也就是这512个字节的MBR是有效的</li>
</ul>
<p>因此在BIOS启动之后，它便会根据设置好的引导顺序，按排位顺序检查引导序列中对应地存储设备是否可以启动 <em>（也就是检查MBR的最后两个字节是否是<code>0x55AA</code>）</em>，如果满足要求，则复制这512字节到内存中的 <strong><code>0x7C00</code></strong> 地址处，然后跳转到该地址开始执行MBR中启动相关的代码</p>
<div class="note info flat"><p>关于为什么MBR是被加载到<code>0x7C00</code>，而不是加载到<code>0x0000</code>，可以参考<a target="_blank" rel="noopener" href="https://www.glamenv-septzen.net/en/view/6">Why BIOS loads MBR into 0x7C00 in x86 ?</a>这篇文章</p>
</div>
</li>
<li>
<p><strong>加载bootloader</strong>：<br>
在CPU跳转到 <strong><code>0x7C00</code></strong> 处之后，就会开始执行MBR中的代码，这部分代码将会由我们编写。在这部分代码中，MBR会从磁盘加载bootloader的代码进入内存，进入保护模式，然后跳转到bootloader处执行</p>
</li>
<li>
<p><strong>加载kernel</strong>：<br>
进入bootloader以后，操作系统会进行相当多准备操作，例如加载文件系统、开启分页机制等，在完成这些工作之后，bootloader会从文件系统中装载系统内核进入内存，然后跳转到内核执行</p>
<p>当进入内核以后，操作系统就基本完成了启动的过程，内核会完成后续操作系统的初始化操作，这些将留到编写内核的时候再详述</p>
</li>
</ol>
<div class="note info flat"><p><strong>启动过程不是绝对的，从MBR之后的启动方式可以按照自己的设计进行</strong><br>
例如，在MBR中可以加载类似grub的代码，在第一阶段加载文件系统，然后从系统路径里加载第二阶段的代码，并且在第二阶段中实现多系统引导，加载对应系统的kernel文件然后跳转运行</p>
</div>
<h3 id="前序知识">前序知识</h3>
<p>在这一章节中将会介绍在后续实验中将会大量使用的基础知识，包括了nasm的语法以及常用概念的解释等</p>
<div class="note info flat"><p><strong>这些内容大部分在用到的时候也会在文章中再次出现</strong><br>
所以如果想要尽快开始实验，不妨跳过这个章节，继续后面的内容</p>
</div>
<h4 id="nasm语法">nasm语法</h4>
<div class="tabs" id="nasm_syntax"><ul class="nav-tabs"><button type="button" data-href="#nasm_syntax-1" class="tab active">寄存器</button><button type="button" data-href="#nasm_syntax-2" class="tab">标识符</button><button type="button" data-href="#nasm_syntax-3" class="tab">标号、变量和常量</button><button type="button" data-href="#nasm_syntax-4" class="tab">寻址</button><button type="button" data-href="#nasm_syntax-5" class="tab">指令</button></ul><div class="tab-contents"><div class="tab-item-content active" id="nasm_syntax-1"><div class="note info flat"><p><strong>IA-32 处理器</strong><br>
IA-32处理器是指从Intel 80386开始到32位的奔腾4处理器，是最为经典的处理器架构<br>
其有三种基本操作模式：保护模式、实地址模式(简称实模式)、系统管理模式和虚拟8086模式<br>
我们在操作系统实验过程中仅用到实模式和保护模式</p>
</div>
<p>IA-32处理器有8个通用寄存器<code>eax</code>, <code>ebx</code>, <code>ecx</code>, <code>edx</code>, <code>ebp</code>, <code>esp</code>, <code>esi</code>, <code>edi</code></p>
<p>除此之外，还有6个段寄存器<code>cs</code>, <code>ss</code>, <code>ds</code>, <code>es</code>, <code>fs</code>, <code>gs</code></p>
<p>以及标志寄存器<code>eflags</code>和指令寄存器<code>eip</code></p>
<div class="note info flat"><p><strong>更多的寄存器</strong><br>
后面的实验中，还会遇到更多寄存器，比如开启保护模式需要用到的<code>cr0</code>，存储页目录表基地址的<code>cr3</code>、存储产生缺页错误的地址的寄存器<code>cr2</code>等<br>
但是目前暂时只需要使用到上面的寄存器</p>
</div>
<p>在<code>nasm</code>中，除了可以直接使用上述的名字访问寄存器的全部内容，还提供了访问部分寄存器低位数据的访问方式：</p>
<table>
<thead>
<tr>
<th style="text-align:center">[31:0]</th>
<th style="text-align:center">[15:0]</th>
<th style="text-align:center">[15:8]</th>
<th style="text-align:center">[7:0]</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center"><code>eax</code></td>
<td style="text-align:center"><code>ax</code></td>
<td style="text-align:center"><code>ah</code></td>
<td style="text-align:center"><code>al</code></td>
</tr>
<tr>
<td style="text-align:center"><code>ebx</code></td>
<td style="text-align:center"><code>bx</code></td>
<td style="text-align:center"><code>bh</code></td>
<td style="text-align:center"><code>bl</code></td>
</tr>
<tr>
<td style="text-align:center"><code>ecx</code></td>
<td style="text-align:center"><code>cx</code></td>
<td style="text-align:center"><code>ch</code></td>
<td style="text-align:center"><code>cl</code></td>
</tr>
<tr>
<td style="text-align:center"><code>edx</code></td>
<td style="text-align:center"><code>dx</code></td>
<td style="text-align:center"><code>dh</code></td>
<td style="text-align:center"><code>dl</code></td>
</tr>
<tr>
<td style="text-align:center"><code>esi</code></td>
<td style="text-align:center"><code>si</code></td>
<td style="text-align:center">–</td>
<td style="text-align:center">–</td>
</tr>
<tr>
<td style="text-align:center"><code>edi</code></td>
<td style="text-align:center"><code>di</code></td>
<td style="text-align:center">–</td>
<td style="text-align:center">–</td>
</tr>
<tr>
<td style="text-align:center"><code>esp</code></td>
<td style="text-align:center"><code>sp</code></td>
<td style="text-align:center">–</td>
<td style="text-align:center">–</td>
</tr>
<tr>
<td style="text-align:center"><code>ebp</code></td>
<td style="text-align:center"><code>bp</code></td>
<td style="text-align:center">–</td>
<td style="text-align:center">–</td>
</tr>
</tbody>
</table>
<p>同时，<code>nasm</code>还有一套约定俗成的规矩，用来指定寄存器的作用</p>
<p>其中，有一些规定是被 <strong>默认使用</strong> 的：</p>
<table>
<thead>
<tr>
<th style="text-align:center">寄存器</th>
<th style="text-align:center">作用</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center"><code>eax</code></td>
<td style="text-align:center">在乘法和除法指令中被自动使用<br>同时也用做返回值寄存器，<code>C</code>或<code>C++</code>函数的返回值放置在<code>eax</code>中</td>
</tr>
<tr>
<td style="text-align:center"><code>ecx</code></td>
<td style="text-align:center">在循环 <em>(loop)</em> 指令中被默认为计数器<br>也即将循环次数存储在<code>ecx</code>中，<code>loop</code>指令只有当<code>ecx</code>为<code>0</code>时停止</td>
</tr>
<tr>
<td style="text-align:center"><code>esp</code></td>
<td style="text-align:center">在对栈进行<code>push</code>和<code>pop</code>操作时，会自动从<code>esp</code>中取基地址<br>也即<code>push eax</code>实际上相当于<code>sub esp, 4</code>，<code>mov eax, [esp]</code></td>
</tr>
<tr>
<td style="text-align:center"><code>esi, edi</code></td>
<td style="text-align:center">用于内存数据高速传送，本次实验中用不到</td>
</tr>
<tr>
<td style="text-align:center"><code>ebp</code></td>
<td style="text-align:center">通常用于在栈上取数据时使用，<strong>一般不用于算数或是数据传输</strong><br>后续<code>C++</code>与<code>asm</code>混编后会经常遇到关于<code>ebp</code>的内容</td>
</tr>
<tr>
<td style="text-align:center"><code>cs</code></td>
<td style="text-align:center"><strong>16位</strong>代码段寄存器</td>
</tr>
<tr>
<td style="text-align:center"><code>ds</code></td>
<td style="text-align:center"><strong>16位</strong>数据段寄存器</td>
</tr>
<tr>
<td style="text-align:center"><code>ss</code></td>
<td style="text-align:center"><strong>16位</strong>栈段寄存器</td>
</tr>
</tbody>
</table>
<p>还有一些类似于习惯的规定，似乎不遵守它们也不会让程序出现错误：</p>
<table>
<thead>
<tr>
<th style="text-align:center">寄存器</th>
<th style="text-align:center">作用</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center"><code>eax</code></td>
<td style="text-align:center">一般用于算数运算<br>比如<code>add eax, 4</code>，因而也叫做累加寄存器</td>
</tr>
<tr>
<td style="text-align:center"><code>ebx</code></td>
<td style="text-align:center">基地址寄存器，用于在取址时提供基地址<br>比如<code>mov eax, dword [ebx + 2]</code>为取出<code>ebx + 2</code>位置上<code>32</code>位的数据放入<code>eax</code></td>
</tr>
<tr>
<td style="text-align:center"><code>edx</code></td>
<td style="text-align:center">数据寄存器</td>
</tr>
<tr>
<td style="text-align:center"><code>esi</code></td>
<td style="text-align:center">源变址寄存器，用于在取值时提供偏移的地址<br>比如<code>mov eax, dword [ebx + esi + 2]</code>或是<code>mov eax, dword [esi + 2]</code></td>
</tr>
<tr>
<td style="text-align:center"><code>edi</code></td>
<td style="text-align:center">目的变址寄存器，用于在取值时提供偏移的地址<br>比如<code>mov dword [edi + 2], eax</code></td>
</tr>
<tr>
<td style="text-align:center"><code>es, fs, gs</code></td>
<td style="text-align:center"><strong>16位</strong>附加段寄存器，用来存放其他的段</td>
</tr>
</tbody>
</table>
<div class="note info flat"><p><strong>寻址</strong><br>
基地址寄存器和变址寄存器涉及到 <strong>寻址</strong> 相关的内容，可以前往寻址页面查看</p>
</div>
<div class="note info flat"><p><strong>段寄存器</strong><br>
段寄存器涉及到 <strong>保护模式</strong> 相关的内容，可以前往<a href="#%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5">基本概念</a>查看</p>
</div><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="nasm_syntax-2"><p>标识符是我们取的名字，用来表示变量、常量、过程、函数或代码标号</p>
<p>标识符满足如下要求和特性：</p>
<ul>
<li>至少1个字符，最多247个字符</li>
<li>大小写不敏感</li>
<li>第一个字符必须是字母、<code>_</code>或是<code>@</code>，后续字符可以包含数字</li>
<li>不能与保留字相同</li>
</ul>
<p>下面是一些有效的标识符</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><span class="line">var1 </span><br><span class="line">Count </span><br><span class="line">_main </span><br><span class="line">MAX </span><br><span class="line">open_file </span><br><span class="line">@myfile</span><br></pre></td></tr></table></figure>
<p>下面是一些使用标识符的示例</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Function</span></span><br><span class="line"><span class="symbol">function_1:</span></span><br><span class="line">  <span class="keyword">ret</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Data</span></span><br><span class="line">count <span class="built_in">dw</span> <span class="number">100</span></span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="nasm_syntax-3"><p>标号是充当指令或数据位置标记的标识符，可以直接将其理解为一个地址值，<strong>指向的是其之后指令或是数据的起始地址</strong></p>
<p>例如</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">count <span class="built_in">dw</span> <span class="number">100</span></span><br></pre></td></tr></table></figure>
<p>中<code>count</code>指向的就是其之后4字节长度的数据<code>100</code>的起始地址</p>
<p>如果有更长的数据，则可以这样写</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">pgdt <span class="built_in">dw</span> <span class="number">0</span></span><br><span class="line">     <span class="built_in">dd</span> <span class="number">0x8800</span></span><br></pre></td></tr></table></figure>
<div class="note info flat"><p><strong>在<code>.asm</code>文件中，可以视为地址是从第一行代码向下增长的</strong></p>
</div>
<p>这样写相当于声明了一个值为<code>0x880000000000</code>的数据</p>
<p>其低4字节为<code>dw 0</code>所声明的值，高2字节为<code>dd 0x8800</code>声明的值</p>
<p>而<code>pgdt</code>指向其低四字节的起始地址，也即该数据最低字节的地址</p>
<div class="note warning flat"><p><strong>IA-32为小端序</strong><br>
在实验中使用的IA-32处理器所使用的是小端序 <em>(little-endian)</em><br>
这意味着数据的低位被放在低地址处而高位被放在高地址处<br>
对于一个多字节的数据，<strong>其标识符指向的是它的最低字节数据地址</strong></p>
</div>
<p>标号也可以用于声明相当于<code>C</code>语言中数组的数据项</p>
<p>数据项之间以<code>,</code>分隔，标号指向第一个数据项最低字节的地址</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">array <span class="built_in">dw</span> <span class="number">1024</span>, <span class="number">2048</span></span><br><span class="line">      <span class="built_in">dw</span> <span class="number">4096</span>, <span class="number">8192</span></span><br></pre></td></tr></table></figure>
<p>上述代码按照<code>1024</code>，<code>2048</code>，<code>4096</code>，<code>8192</code>的顺序存放数据，同时<code>array</code>指向<code>1024</code></p>
<p>标号也可以用来在过程中添加标记以方便跳转</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br></pre></td><td class="code"><pre><span class="line"><span class="symbol">Stage_1:</span></span><br><span class="line">  <span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">ebx</span></span><br><span class="line">  <span class="comment">; 省略其他步骤</span></span><br><span class="line">  <span class="keyword">jmp</span> Stage_1 <span class="comment">; 跳转到 Stage_1 处执行</span></span><br></pre></td></tr></table></figure>
<p>这里，<code>Stage_1</code>并不会编译成汇编代码，而是会编译为<code>mov eax, ebx</code>这一步的地址值，<code>jmp Stage_1</code>相当于跳转到<code>mov eax, ebx</code>这句执行</p>
<div class="note info flat"><p><strong>标号后面的换行和空格并不会影响标号的值</strong><br>
也就是说<code>Stage_1: mov eax, ebx</code>中<code>Stage_1</code>跟上面的代码拥有一样的标号值</p>
</div><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="nasm_syntax-4"><div class="note info flat"><p><strong>如何理解寻址</strong><br>
寻址可以理解为代码获得某个数据的方式<br>
例如，<strong>寄存器寻址</strong> 可以理解为代码通过寄存器获得数据</p>
</div>
<p><code>nasm</code>有六种寻址方式，分别为</p>
<ul>
<li>
<p><strong>寄存器寻址</strong>：操作数存放在寄存器中，从寄存器中取得数据</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">ebx</span> <span class="comment">; eax = ebx</span></span><br></pre></td></tr></table></figure>
</li>
<li>
<p><strong>立即数寻址</strong>：直接将立即数作为操作数</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="number">0x20</span> <span class="comment">; eax = 0x20</span></span><br></pre></td></tr></table></figure>
</li>
<li>
<p><strong>直接寻址</strong>：从立即数或是标号指向的地址处取得数据</p>
<p>从立即数指向的地址取得数据</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; 从0x5C00处取得数据</span></span><br><span class="line"><span class="comment">; </span></span><br><span class="line"><span class="comment">; 相当于C语言中</span></span><br><span class="line"><span class="comment">; uint32* ptr = (uint32*)0x5C00;</span></span><br><span class="line"><span class="comment">; eax = *ptr</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [<span class="number">0x5C00</span>]</span><br></pre></td></tr></table></figure>
<p>也可以从标号指向的地址取得数据</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">tag <span class="built_in">dw</span> <span class="number">0x20</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [tag]  <span class="comment">; eax = 0x20</span></span><br></pre></td></tr></table></figure>
</li>
<li>
<p><strong>基址寻址</strong>：将基址寄存器所储存的数值视为地址并从该地址处取得数据</p>
<p>基址寻址类似于数组的寻址，基址寄存器只能是寄存器<code>bx</code>或<code>bp</code></p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line">tag <span class="built_in">dw</span> <span class="number">0x20</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ebx</span>, tag  <span class="comment">; ebx中存储的是tag的值，也就是0x20的地址</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [<span class="built_in">ebx</span>]  <span class="comment">; eax = 0x20</span></span><br></pre></td></tr></table></figure>
<p>基址寻址也可以使用基址寄存器和立即数来构成真实的偏移地址</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><span class="line">tag <span class="built_in">dw</span> <span class="number">0x20</span></span><br><span class="line"><span class="comment">; 构造一个与tag对应的地址差4的地址存入ebx</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ecx</span>, tag</span><br><span class="line"><span class="keyword">sub</span> <span class="built_in">ecx</span>, <span class="number">4</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ebx</span>, <span class="built_in">ecx</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [<span class="built_in">ebx</span> + <span class="number">4</span>]  <span class="comment">; eax = 0x20</span></span><br></pre></td></tr></table></figure>
</li>
<li>
<p><strong>变址寻址</strong>：使用变址寄存器和立即数来构成真实的偏移地址，从该地址处取得数据</p>
<p>变址寻址与基址寻址相似，变址寄存器只能是<code>si</code>或<code>di</code></p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [<span class="built_in">esi</span> + <span class="number">4</span> * <span class="number">4</span>] <span class="comment">; eax = *(uint32*)(esi + 4 * 4)</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [<span class="built_in">edi</span>], <span class="number">0x5</span> <span class="comment">; *(uint32*)edi = 0x5</span></span><br></pre></td></tr></table></figure>
</li>
<li>
<p><strong>基址变址寻址</strong>：通过基址寄存器、变址寄存器、立即数来构成真实的偏移地址，从该地址处取得数据</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [<span class="built_in">ebx</span> + <span class="built_in">esi</span> + <span class="number">4</span> * <span class="number">5</span>] <span class="comment">; eax = *(uint32*)(ebx + esi + 4 * 5)</span></span><br></pre></td></tr></table></figure>
</li>
</ul>
<div class="note orange icon-padding flat"><i class="note-icon fas fa-question-circle"></i><p><strong>不够用的地址？</strong><br>
在 <strong>实模式</strong> 下，IA-32处理器使用 <strong>20位</strong> 的地址线<br>
不难发现在这种情况下可以访问的内存范围为<span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><msup><mn>2</mn><mn>20</mn></msup><mo>=</mo><mn>1</mn><mi>M</mi><mi>B</mi></mrow><annotation encoding="application/x-tex">2^{20} = 1MB</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.8141em;"></span><span class="mord"><span class="mord">2</span><span class="msupsub"><span class="vlist-t"><span class="vlist-r"><span class="vlist" style="height:0.8141em;"><span style="top:-3.063em;margin-right:0.05em;"><span class="pstrut" style="height:2.7em;"></span><span class="sizing reset-size6 size3 mtight"><span class="mord mtight"><span class="mord mtight">20</span></span></span></span></span></span></span></span></span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.6833em;"></span><span class="mord">1</span><span class="mord mathnormal" style="margin-right:0.05017em;">MB</span></span></span></span>，范围从<code>0x0000</code>到<code>0xFFFF</code><br>
但是寄存器只有 <strong>16位</strong>，意味着使用基址、变址寻址的时候没办法访问到所有的内存地址</p>
</div>
<p>为了解决这个问题，工程师提出了 <strong>段</strong> 的概念，使用了段地址之后，实际的地址可以表示为</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi><mo>=</mo><mo stretchy="false">(</mo><mi>S</mi><mi>e</mi><mi>g</mi><mi>m</mi><mi>e</mi><mi>n</mi><mi>t</mi><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi><mo>&lt;</mo><mo>&lt;</mo><mn>4</mn><mo stretchy="false">)</mo><mo>+</mo><mi>S</mi><mi>h</mi><mi>i</mi><mi>f</mi><mi>t</mi><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi></mrow><annotation encoding="application/x-tex">Addr = (SegmentAddr &lt;&lt; 4) + ShiftAddr
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.6944em;"></span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">e</span><span class="mord mathnormal" style="margin-right:0.03588em;">g</span><span class="mord mathnormal">m</span><span class="mord mathnormal">e</span><span class="mord mathnormal">n</span><span class="mord mathnormal">t</span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">&lt;&lt;</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mord">4</span><span class="mclose">)</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">hi</span><span class="mord mathnormal" style="margin-right:0.10764em;">f</span><span class="mord mathnormal">t</span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span></span></span></span></span></p>
<p>也就是物理地址由左移4位的段地址加上偏移地址组成，这个偏移地址就由标号或是寄存器提供</p>
<div class="note warning flat"><p><strong>此段非彼段</strong><br>
目前我们讨论的是 <strong>实模式</strong> 下的问题，此时的段是用来解决寄存器不足以访问所有的内存地址的问题<br>
在后面的 <strong>保护模式</strong> 中，还将看到另一个段定义，它与本节所述的段并不是同一个概念</p>
</div>
<p>段地址的使用一般出现在使用<code>[]</code>来寻址的操作中，因为在这种情况下<code>[]</code>内表示的是一个地址，而这个地址很可能由一个16位的寄存器或是标号提供</p>
<p>添加了段地址的寻址语法为</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [<span class="built_in">ds</span>:<span class="built_in">bx</span>]</span><br></pre></td></tr></table></figure>
<p>在默认的情况下，系统会根据使用的标号和寄存器自动指定段地址，指定的段地址与使用的基址寄存器有直接关系</p>
<div class="note info flat"><p><strong>由于这里讨论的是实模式，所以只讨论16位寄存器</strong></p>
</div>
<p>对于<code>bx</code>、<code>si</code>、<code>di</code>，默认的段寄存器为<code>ds</code></p>
<p>对于<code>bp</code>，默认的段寄存器为<code>ss</code></p>
<p>对于基址变址寻址方式，默认的段寄存器以基址寄存器的使用为准</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; 下面的语句等价</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [tag]</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [<span class="built_in">ds</span>:tag]</span><br><span class="line"></span><br><span class="line"><span class="comment">; 下面的语句等价</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [<span class="built_in">bx</span>]</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [<span class="built_in">ds</span>:<span class="built_in">bx</span>]</span><br><span class="line"></span><br><span class="line"><span class="comment">; 下面的语句等价</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [<span class="built_in">bp</span> + <span class="built_in">si</span> + <span class="number">4</span> * <span class="number">4</span>]</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [<span class="built_in">ss</span>:<span class="built_in">bp</span> + <span class="built_in">si</span> + <span class="number">4</span> * <span class="number">4</span>]</span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="nasm_syntax-5"><p>下文中可能用到的标识符解释如下</p>
<table>
<thead>
<tr>
<th style="text-align:center">标识符</th>
<th style="text-align:center">意义</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">reg</td>
<td style="text-align:center">寄存器</td>
</tr>
<tr>
<td style="text-align:center">imm</td>
<td style="text-align:center">立即数</td>
</tr>
<tr>
<td style="text-align:center">immX</td>
<td style="text-align:center">X位立即数</td>
</tr>
<tr>
<td style="text-align:center">tag</td>
<td style="text-align:center">标号</td>
</tr>
<tr>
<td style="text-align:center">mem</td>
<td style="text-align:center">使用寻址方式取得的内存数据，例如<code>[ebx]</code></td>
</tr>
</tbody>
</table>
<div class="tabs" id="instructions"><ul class="nav-tabs"><button type="button" data-href="#instructions-1" class="tab active">mov</button><button type="button" data-href="#instructions-2" class="tab">add/sub</button><button type="button" data-href="#instructions-3" class="tab">imul/idiv</button><button type="button" data-href="#instructions-4" class="tab">shl/shr</button><button type="button" data-href="#instructions-5" class="tab">inc/dec</button><button type="button" data-href="#instructions-6" class="tab">逻辑操作</button><button type="button" data-href="#instructions-7" class="tab">跳转指令</button><button type="button" data-href="#instructions-8" class="tab">push/pop</button><button type="button" data-href="#instructions-9" class="tab">in/out</button></ul><div class="tab-contents"><div class="tab-item-content active" id="instructions-1"><figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">mov</span> [dest], [src]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">mov</span> &lt;reg&gt;, &lt;reg/imm&gt;</span><br><span class="line"><span class="keyword">mov</span> &lt;reg&gt;, &lt;size&gt; [&lt;reg/imm/tag&gt;]</span><br><span class="line"><span class="keyword">mov</span> &lt;size&gt; [&lt;reg/imm/tag&gt;], &lt;reg/imm&gt;</span><br><span class="line"></span><br><span class="line"><span class="comment">; Example</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">ebx</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="number">0x20</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [<span class="built_in">ebx</span>]</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [<span class="number">0x8800</span>]</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [tag]</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [<span class="built_in">eax</span>], <span class="built_in">ebx</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [<span class="built_in">eax</span>], <span class="number">0x20</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [<span class="number">0x8800</span>], <span class="built_in">ebx</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [<span class="number">0x8800</span>], <span class="number">0x20</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [tag], <span class="built_in">ebx</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [tag], <span class="number">0x20</span></span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="instructions-2"><figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">add</span>/<span class="keyword">sub</span> [dest &amp; op1], [op2]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">add</span>/<span class="keyword">sub</span> &lt;reg&gt;, &lt;reg/imm/mem&gt;</span><br><span class="line"><span class="keyword">add</span>/<span class="keyword">sub</span> &lt;mem&gt;, &lt;reg/imm&gt;</span><br><span class="line"></span><br><span class="line"><span class="comment">; Example</span></span><br><span class="line"><span class="keyword">add</span> <span class="built_in">eax</span>, <span class="built_in">ebx</span></span><br><span class="line"><span class="keyword">sub</span> <span class="built_in">eax</span>, <span class="number">0x20</span></span><br><span class="line"><span class="keyword">add</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [tag]</span><br><span class="line"><span class="keyword">sub</span> <span class="built_in">dword</span> [tag], <span class="built_in">ebx</span></span><br><span class="line"><span class="keyword">add</span> <span class="built_in">dword</span> [tag], <span class="number">0x8800</span></span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="instructions-3"><p><code>imul</code>完成整数乘法操作</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">imul</span> [dest &amp; op1], [op2]</span><br><span class="line"><span class="keyword">imul</span> [dest], [op1], [op2]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">imul</span> &lt;reg&gt;, &lt;reg/mem&gt;</span><br><span class="line"><span class="keyword">imul</span> &lt;reg&gt;, &lt;reg/mem&gt;, &lt;imm&gt;</span><br><span class="line"></span><br><span class="line"><span class="comment">; Example</span></span><br><span class="line"><span class="keyword">imul</span> <span class="built_in">eax</span>, <span class="built_in">ebx</span></span><br><span class="line"><span class="keyword">imul</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [var]</span><br><span class="line"><span class="keyword">imul</span> <span class="built_in">esi</span>, <span class="built_in">edi</span>, <span class="number">25</span></span><br><span class="line"><span class="keyword">imul</span> <span class="built_in">esi</span>, <span class="built_in">dword</span> [shift], <span class="number">4</span></span><br></pre></td></tr></table></figure>
<p><code>idiv</code>完成整数除法操作</p>
<p><code>idiv</code>只有一个操作数，此操作数为除数，而被除数则为<code>edx:eax</code>中的内容 <em>(一个64位的整数)</em></p>
<p>操作的结果有两个部分</p>
<ul>
<li><strong>商</strong>：存放在<code>eax</code>寄存器中</li>
<li><strong>余数</strong>：存放在<code>edx</code>寄存器中</li>
</ul>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">idiv</span> [divisor]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">idiv</span> &lt;reg/mem&gt;</span><br><span class="line"></span><br><span class="line"><span class="comment">; Example</span></span><br><span class="line"><span class="keyword">idiv</span> <span class="built_in">ebx</span></span><br><span class="line"><span class="keyword">idiv</span> <span class="built_in">dword</span> [var]</span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="instructions-4"><p><code>shl</code>，<code>shr</code>表示逻辑左移和逻辑右移，空出的位补<code>0</code></p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">shl</span>/<span class="keyword">shr</span> [target], [shift]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">shl</span>/<span class="keyword">shr</span> &lt;reg/mem&gt;, &lt;con&gt;</span><br><span class="line"><span class="keyword">shl</span>/<span class="keyword">shr</span> &lt;reg/mem&gt;, <span class="built_in">cl</span> <span class="comment">; cl为ecx低8位</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Example</span></span><br><span class="line"><span class="keyword">shl</span> <span class="built_in">eax</span>, <span class="number">5</span></span><br><span class="line"><span class="keyword">shr</span> <span class="built_in">dword</span> [<span class="built_in">ebx</span>], <span class="number">4</span></span><br><span class="line"><span class="keyword">shl</span> <span class="built_in">eax</span>, <span class="built_in">cl</span></span><br><span class="line"><span class="keyword">shr</span> dowrd [<span class="built_in">ebx</span>], <span class="built_in">cl</span></span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="instructions-5"><p><code>inc</code>，<code>dec</code>指令分别表示自增<code>1</code>或自减<code>1</code></p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">inc</span>/<span class="keyword">dec</span> [target]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">inc</span>/<span class="keyword">dec</span> &lt;reg/mem&gt;</span><br><span class="line"></span><br><span class="line"><span class="comment">; Example</span></span><br><span class="line"><span class="keyword">inc</span> <span class="built_in">eax</span></span><br><span class="line"><span class="keyword">dec</span> <span class="built_in">byte</span> [tag]</span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="instructions-6"><p><code>and</code>, <code>or</code>, <code>xor</code>分别表示将两个操作数逻辑与、逻辑或和逻辑异或后放入到第一个操作数中</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">and</span>/<span class="keyword">or</span>/<span class="keyword">xor</span> [dest &amp; op1], [op2]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">and</span>/<span class="keyword">or</span>/<span class="keyword">xor</span> &lt;reg/mem&gt;, &lt;reg/imm&gt;</span><br><span class="line"><span class="keyword">and</span>/<span class="keyword">or</span>/<span class="keyword">xor</span> &lt;reg&gt;, &lt;mem&gt;</span><br></pre></td></tr></table></figure>
<p><code>not</code>表示对操作数每一位取反，<code>neg</code>表示对操作数取负</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">not</span> [target]</span><br><span class="line"><span class="keyword">neg</span> [target]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">not</span> &lt;reg/mem&gt;</span><br><span class="line"><span class="keyword">neg</span> &lt;reg/mem&gt;</span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="instructions-7"><p><code>jmp</code>指令是无条件跳转指令，跳转到代码标号的指令处执行</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">jmp</span> [addr]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">jmp</span> &lt;tag&gt;</span><br><span class="line"></span><br><span class="line"><span class="comment">; Example</span></span><br><span class="line"><span class="keyword">jmp</span> func_1</span><br></pre></td></tr></table></figure>
<p>除了无条件跳转指令外，还有条件跳转指令</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">je</span> &lt;tag&gt;   <span class="comment">; jump when equal</span></span><br><span class="line"><span class="keyword">jne</span> &lt;tag&gt;  <span class="comment">; jump when not equal</span></span><br><span class="line"><span class="keyword">jz</span> &lt;tag&gt;   <span class="comment">; jump when last result was zero</span></span><br><span class="line"><span class="keyword">jg</span> &lt;tag&gt;   <span class="comment">; jump when greater than</span></span><br><span class="line"><span class="keyword">jge</span> &lt;tag&gt;  <span class="comment">; jump when greater than or equal to</span></span><br><span class="line"><span class="keyword">jl</span> &lt;tag&gt;   <span class="comment">; jump when less than</span></span><br><span class="line"><span class="keyword">jle</span> &lt;tag&gt;  <span class="comment">; jump when less than or equal to</span></span><br></pre></td></tr></table></figure>
<p>在使用条件跳转指令之前，要先进行判断，判断使用的是<code>cmp</code>指令</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">cmp</span> [op1], [op2]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">cmp</span> &lt;reg/mem&gt;, &lt;reg/imm&gt;</span><br><span class="line"><span class="keyword">cmp</span> &lt;reg&gt;, &lt;mem&gt;</span><br><span class="line"></span><br><span class="line"><span class="comment">; Example</span></span><br><span class="line"><span class="keyword">cmp</span> <span class="built_in">eax</span>, <span class="number">10</span></span><br></pre></td></tr></table></figure>
<p>一个条件跳转的示例如下，其展示了一个不适用<code>loop</code>指令循环10次的简单实现</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">xor</span> <span class="built_in">eax</span>, <span class="built_in">eax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="number">10</span></span><br><span class="line"><span class="symbol">loop_start:</span></span><br><span class="line">  <span class="keyword">dec</span> <span class="built_in">eax</span></span><br><span class="line">  <span class="keyword">cmp</span> <span class="built_in">eax</span>, <span class="number">0</span></span><br><span class="line">  <span class="keyword">je</span> exit</span><br><span class="line">  <span class="keyword">jmp</span> loop_start</span><br><span class="line"><span class="symbol">exit:</span></span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="instructions-8"><p><code>push</code>和<code>pop</code>为栈操作，负责将寄存器值压栈、弹栈</p><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="instructions-9"><div class="note icon-padding flat"><i class="note-icon fa fa-external-link-square"></i><p><strong>本页的内容基于<a target="_blank" rel="noopener" href="https://c9x.me/x86/html/file_module_x86_id_139.html">x86 Instruction Set Reference中关于in指令的介绍</a>以及<a target="_blank" rel="noopener" href="https://c9x.me/x86/html/file_module_x86_id_222.html">x86 Instruction Set Reference中关于out指令的介绍</a></strong></p>
</div>
<p><code>in</code>、<code>out</code>为端口读写操作，它们的操作数非常严格</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Syntax</span></span><br><span class="line"><span class="keyword">in</span> [dest], [src]  <span class="comment">; Input byte from [src] port to [dest] register</span></span><br><span class="line"><span class="keyword">out</span> [dest], [src] <span class="comment">; Output byte from [src] register to [dest] port</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Usage</span></span><br><span class="line"><span class="keyword">in</span> <span class="built_in">al</span>/<span class="built_in">ax</span>/<span class="built_in">eax</span>, &lt;imm8/<span class="built_in">dx</span>&gt;   <span class="comment">; imm8 means a byte immediate</span></span><br><span class="line"><span class="keyword">out</span> &lt;imm8/<span class="built_in">dx</span>&gt;, <span class="built_in">al</span>/<span class="built_in">ax</span>/<span class="built_in">eax</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Example</span></span><br><span class="line"><span class="keyword">in</span> <span class="built_in">al</span>, <span class="number">0x21</span>   <span class="comment">; Input byte from 0x21 port to al</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x379</span> <span class="comment">; Set dx as port 0x379</span></span><br><span class="line"><span class="keyword">in</span> <span class="built_in">al</span>, <span class="built_in">dx</span>     <span class="comment">; Input byte from 0x379 port to al</span></span><br><span class="line"></span><br><span class="line"><span class="keyword">out</span> <span class="number">0x21</span>, <span class="built_in">al</span>  <span class="comment">; Output byte from al to 0x21 port</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x378</span> <span class="comment">; Set dx as port 0x378</span></span><br><span class="line"><span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span>    <span class="comment">; Output byte from al to port 0x378</span></span><br></pre></td></tr></table></figure>
<p>其中，<code>al/ax/eax</code>根据端口位宽设置，如果对8位端口指定了16位输入/输出，则会连带该端口的下一个端口一并进行输入/输出</p><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div></div></div><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div></div></div>
<h4 id="基本概念">基本概念</h4>
<div class="tabs" id="concepts"><ul class="nav-tabs"><button type="button" data-href="#concepts-1" class="tab active">实模式</button><button type="button" data-href="#concepts-2" class="tab">保护模式</button><button type="button" data-href="#concepts-3" class="tab">中断</button></ul><div class="tab-contents"><div class="tab-item-content active" id="concepts-1"><div class="note icon-padding flat"><i class="note-icon fa fa-external-link-square"></i><p><strong>本页的内容基于<a target="_blank" rel="noopener" href="https://wiki.osdev.org/Real_Mode">OSDev Wiki</a>关于实模式的介绍</strong></p>
</div>
<p>实模式是所有的<code>x86</code>处理器都有的一个16位运行模式</p>
<p>这个模式是为早期的操作系统设计的，它的出现远远早于保护模式</p>
<p>即使现在的操作系统已经运行在保护模式下了，但是处于兼容性考虑，所有的现代操作系统都需要先从实模式开始运行，然后再切换到保护模式</p>
<p>在实模式下，CPU只有20根地址线可用，也即可用内存为</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><msup><mn>2</mn><mn>20</mn></msup><mi>B</mi><mi>y</mi><mi>t</mi><mi>e</mi><mo>=</mo><mn>1</mn><mi>M</mi><mi>i</mi><mi>B</mi></mrow><annotation encoding="application/x-tex">2^{20}Byte = 1MiB
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:1.0585em;vertical-align:-0.1944em;"></span><span class="mord"><span class="mord">2</span><span class="msupsub"><span class="vlist-t"><span class="vlist-r"><span class="vlist" style="height:0.8641em;"><span style="top:-3.113em;margin-right:0.05em;"><span class="pstrut" style="height:2.7em;"></span><span class="sizing reset-size6 size3 mtight"><span class="mord mtight"><span class="mord mtight">20</span></span></span></span></span></span></span></span></span><span class="mord mathnormal" style="margin-right:0.05017em;">B</span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span><span class="mord mathnormal">t</span><span class="mord mathnormal">e</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.6833em;"></span><span class="mord">1</span><span class="mord mathnormal" style="margin-right:0.10903em;">M</span><span class="mord mathnormal">i</span><span class="mord mathnormal" style="margin-right:0.05017em;">B</span></span></span></span></span></p>
<p>为了让16位寄存器能够对全部内存地址空间进行取址，额外引入了段寄存器<code>cs</code>、<code>ss</code>、<code>ds</code>和<code>es</code>、<code>fs</code>、<code>gs</code>，地址的表示由段地址和寄存器表示的偏移地址组合而成</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mi>R</mi><mi>e</mi><mi>a</mi><mi>l</mi><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi><mi>e</mi><mi>s</mi><mi>s</mi><mo>=</mo><mo stretchy="false">(</mo><mi>S</mi><mi>e</mi><mi>g</mi><mi>m</mi><mi>e</mi><mi>n</mi><mi>t</mi><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi><mi>e</mi><mi>s</mi><mi>s</mi><mo>&lt;</mo><mo>&lt;</mo><mn>4</mn><mo stretchy="false">)</mo><mo>+</mo><mi>S</mi><mi>h</mi><mi>i</mi><mi>f</mi><mi>t</mi><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi><mi>e</mi><mi>s</mi><mi>s</mi></mrow><annotation encoding="application/x-tex">RealAddress = (SegmentAddress &lt;&lt; 4) + ShiftAddress
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.6944em;"></span><span class="mord mathnormal" style="margin-right:0.00773em;">R</span><span class="mord mathnormal">e</span><span class="mord mathnormal">a</span><span class="mord mathnormal" style="margin-right:0.01968em;">l</span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal">ress</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">e</span><span class="mord mathnormal" style="margin-right:0.03588em;">g</span><span class="mord mathnormal">m</span><span class="mord mathnormal">e</span><span class="mord mathnormal">n</span><span class="mord mathnormal">t</span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal">ress</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">&lt;&lt;</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mord">4</span><span class="mclose">)</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">hi</span><span class="mord mathnormal" style="margin-right:0.10764em;">f</span><span class="mord mathnormal">t</span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal">ress</span></span></span></span></span></p>
<p>段地址保存在段寄存器中，其中<code>es</code>、<code>fs</code>和<code>gs</code>为附加段寄存器，可以由用户自行指定</p><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="concepts-2"><div class="note icon-padding flat"><i class="note-icon fa fa-external-link-square"></i><p><strong>本页的内容基于<a target="_blank" rel="noopener" href="https://wiki.osdev.org/Protected_Mode">OSDev Wiki</a>关于保护模式的介绍</strong></p>
</div>
<p>保护模式是目前英特尔处理器主流的运行模式，在保护模式中，处理器以32位模式运行，所有的寄存器也都为32位</p>
<p>为了进入保护模式，操作系统需要在实模式下进行一系列操作：</p>
<ul>
<li>设置全局描述符表</li>
<li>关闭中断</li>
<li>开启第21根地址线</li>
<li>打开保护模式开关</li>
<li>执行一次远跳转送入代码段选择子</li>
</ul>
<p>具体的切换方式让我们留到实验中再行讲解</p>
<p>保护模式之所以叫做保护模式，因为其引入了 <strong>“段”</strong> 的概念，每一个程序都有它自己的段，一旦程序错误地访问其他段的地址空间，那么CPU就会产生异常</p>
<div class="note info flat"><p><strong>段</strong> <em>(segment)</em><br>
段实际上是程序员人为划分的一块块连续的内存区域，或者称为地址空间</p>
</div>
<div class="note warning flat"><p><strong>误区</strong><br>
这里的段概念与实模式的段概念并不相同</p>
</div>
<p>也就是说，可以认为保护模式保护的是地址空间，防止程序代码错误地访问了非它自己的段地址空间，造成越界访问</p>
<p>为了让CPU知道段的范围，工程师引入了<strong>全局描述符表</strong>的概念</p>
<p>全局描述符表可以理解为<strong>一段连续的内存</strong>，类似于数组，<strong>其中存储了全局描述符</strong>，每个描述符都对应了一个段的设置</p>
<p>为了取出对应的全局描述符，就如同用下标从数组中获取数据那样，<strong>CPU使用选择子从全局描述符表中获取描述符</strong></p>
<p>选择子除了包含了取出描述符所需要的下标信息，还包含了一些权限信息等，这些<strong>选择子会被存储在段寄存器中</strong></p>
<div class="note info flat"><p><strong>更多的保护内容</strong><br>
除了段保护之外，保护模式实际上还包括了特权级的保护、页保护等<br>
例如，保护模式会阻值低特权级代码访问高特权级段空间，会依据特权级限制程序操作等</p>
</div><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="concepts-3"><div class="note grey icon-padding flat"><i class="note-icon fas fa-quote-right"></i><p><strong>操作系统是由中断驱动的</strong></p>
</div>
<p>操作系统的最终目的是完成用户的任务，但很显然，用户不会时时刻刻都有需要操作系统完成的任务，同时，当用户没有向操作系统提交任务的时候，它应当去干点别的而不是傻傻地待在原地等着用户提交任务——这也就是中断的意义</p>
<p>可以把操作系统想象成一个忙碌的打工人，当你向他提交了任务以后，他便会开始完成你提交的任务。但如果在他进行到一半的时候，你突然又想让他干点别的，该怎么办呢？对了！就是拍拍他的肩膀，说：“伙计，先停停手上的活，把这件事情做了，然后再回来做现在的活儿。”</p>
<p>中断对于操作系统的作用也正是这样，通过中断，可以让CPU暂停当前正在运行的代码，转而对产生中断的信息进行处理，当处理完后再返回运行原先的代码</p>
<p>通过中断，可以让操作系统在等待用户/内核打断的同时进行其他的任务，而不需要在原地循环等待用户的下一步指令</p>
<p>又由于操作系统实质就是无数个 <strong>中断-处理-返回</strong> 的过程，所以可以说这就是操作系统进行任务的最根本方式，所有的操作都要经由中断来实施，所以可以说 <em>“操作系统是由中断驱动的”</em></p><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div></div></div>
<h3 id="实验路线">实验路线</h3>
<div class="timeline undefined"><div class='timeline-item headline'><div class='timeline-item-title'><div class='item-circle'><p>第零阶段：知识准备</p>
</div></div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>环境搭建</strong></p>
</div></div><div class='timeline-item-content'><p>在这一步中，需要搭建实验所必需的<code>Linux</code>环境以及虚拟机<code>qemu</code>和<code>C</code>、<code>C++</code>、<code>asm</code>编译器等</p>
<div class="note success flat"><p><strong>如果读完了 <a href="https://blog.linloir.cn/2022/07/15/os-journal-vol-1/index.html">操作系统日志 第一章</a> 的话，那么这一步已经完成了</strong></p>
</div>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>汇编语法与操作系统概念</strong></p>
</div></div><div class='timeline-item-content'><p>由于实验的第一部分 <em>( 编写MBR )</em> 就需要使用到汇编语法，因此首先需要了解的就是<code>nasm</code>汇编语法</p>
<p>其次，为了能够对实验有整体的把握，建议先完整阅读 <strong>《 操作系统概念（第九版）》</strong> 对操作系统有一个整体视角再着手开始实验，这样更有利于落实自己的想法并体验到实验的乐趣所在</p>
<p>当然，如果是正在跟随课程实验进行，尚不能提前通读整本书的话，也不要担心，因为随着实验的进行也是完全能够循序渐进地认识操作系统的各个概念的</p>
<div class="note success flat"><p><strong>如果阅读了前文的 <a href="#%E5%89%8D%E5%BA%8F%E7%9F%A5%E8%AF%86">前序知识</a> 的话，那么就已经足够可以让我们编写MBR了，让我们继续吧</strong></p>
</div>
</div></div></div>
<div class="timeline undefined"><div class='timeline-item headline'><div class='timeline-item-title'><div class='item-circle'><p>第一阶段：启动Kernel</p>
</div></div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>MBR</strong></p>
</div></div><div class='timeline-item-content'><p>本节需要完成：</p>
<ul>
<li>使用 <strong>中断</strong> 加载<code>bootloader</code></li>
<li>跳转到<code>bootloader</code>运行</li>
</ul>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>bootloader</strong></p>
</div></div><div class='timeline-item-content'><p>本节需要完成：</p>
<ul>
<li>切换到 <strong>保护模式</strong></li>
<li>从磁盘加载<code>kernelloader</code></li>
<li>跳转到<code>kernelloader</code>运行</li>
</ul>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>准备</strong></p>
</div></div><div class='timeline-item-content'><div class="note info flat"><p><strong>从这一节开始，将会从<code>asm</code>编程切换到<code>C++</code>和内联汇编组合编程</strong></p>
</div>
<p>为了能够使得操作系统能够方便地组织和使用硬件资源，需要对硬件的接口进行 <strong>抽象</strong>，编写合适的 <strong>驱动</strong></p>
<p>同时，为了加强代码的可读性，需要将常用的结构包装为 <strong>类</strong></p>
<div class="note info flat"><p><strong>驱动的编写和结构的封装并不需要在这一步就全部完成，而是随着后续代码的编写而逐渐完善</strong></p>
</div>
<p>为了后续实验顺利进行，在这一步需要完成如下驱动：</p>
<ul>
<li>端口驱动</li>
<li><code>UART</code>驱动</li>
<li>磁盘驱动</li>
<li>显示驱动</li>
</ul>
<p>同时需要完成如下结构体：</p>
<ul>
<li><code>pagetable</code>：页表</li>
<li><code>page</code>：页</li>
<li><code>frame</code>：帧</li>
<li><code>elf head</code>：<code>ELF</code>头</li>
</ul>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>kernelloader</strong></p>
</div></div><div class='timeline-item-content'><p>本节需要完成：</p>
<ul>
<li>开启 <strong>分页机制</strong></li>
<li>装载 <strong>文件系统</strong></li>
<li>编写<code>elf parser</code></li>
<li>从文件系统中读取<code>kernel.o</code></li>
<li>使用<code>elf parser</code>解析<code>kernel.o</code>中的 <strong>ELF头</strong> 并加载内核入内存正确地址处</li>
<li>跳转到内核运行</li>
</ul>
<div class="note blue icon-padding flat"><i class="note-icon fa fa-external-link-square"></i><p><strong>想要支持多系统？</strong><br>
在<code>linux</code>的实现中，多系统通过<code>grub</code>提供引导，关于<code>grub</code>的更多信息可以查看 <a target="_blank" rel="noopener" href="https://zh.wikipedia.org/zh-hans/GNU_GRUB">GNU GRUB的Wiki页面</a></p>
</div>
</div></div></div>
<div class="timeline undefined"><div class='timeline-item headline'><div class='timeline-item-title'><div class='item-circle'><p>第二阶段：内核态</p>
</div></div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>堆分配器</strong></p>
</div></div><div class='timeline-item-content'><p>实现<code>malloc</code>和<code>free</code></p>
<div class="note success flat"><p><strong>从此节开始后，将可以使用<code>malloc</code>和<code>free</code>函数</strong><br>
<strong>这也意味着同样也可以使用<code>vector</code>等等各种数据结构了，代码的编写自此进入现代化</strong></p>
</div>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>页帧分配器</strong></p>
</div></div><div class='timeline-item-content'><p>实现 <strong>物理页帧</strong> 的管理和分配</p>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>虚拟页管理器</strong></p>
</div></div><div class='timeline-item-content'><p>实现 <strong>虚拟页</strong> 的管理和分配，实现 <strong>虚拟页到物理页的映射</strong></p>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>描述符表</strong></p>
</div></div><div class='timeline-item-content'><p>实现<code>GDT</code>、<code>IDT</code>和<code>TSS</code>的抽象，并初始化描述符表</p>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>中断</strong></p>
</div></div><div class='timeline-item-content'><p>实现 <strong>中断处理函数</strong></p>
<div class="note warning flat"><p><strong>本节中会涉及部分用户进程的知识</strong><br>
由于用户进程和内核进程都需要使用中断，同时用户进程和内核进程进入中断时在栈上的行为还<s>天杀的</s>不一样<br>
为了避免反复编写重复代码，会提前介绍用户进程的行为以及如何编写代码解决其中的问题</p>
</div>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>内核进程</strong></p>
</div></div><div class='timeline-item-content'><p>实现 <strong>内核进程</strong> 以及 <strong>进程管理器</strong></p>
<div class="note success flat"><p><strong>完成本节后，就有了进程调度机制，可以支持并发了</strong></p>
</div>
</div></div></div>
<div class="timeline undefined"><div class='timeline-item headline'><div class='timeline-item-title'><div class='item-circle'><p>第三阶段：用户态</p>
</div></div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>系统调用</strong></p>
</div></div><div class='timeline-item-content'><p>由于用户进程不能直接运行内核相关的代码，因此需要实现系统调用来在对用户进程透明的情况下执行内核相关代码</p>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>用户进程</strong></p>
</div></div><div class='timeline-item-content'><p>在完成系统调用之后，就可以实现 <strong>用户进程</strong> 了</p>
</div></div><div class='timeline-item'><div class='timeline-item-title'><div class='item-circle'><p><strong>Shell</strong></p>
</div></div><div class='timeline-item-content'><p>实现一个 <strong>Shell</strong> 程序提供用户交互</p>
</div></div></div>
<h2 id="从MBR开始编写自己的操作系统">从MBR开始编写自己的操作系统</h2>
<div class="note success flat"><p><strong>真正的实验从这里开始</strong></p>
</div>
<h3 id="创建项目">创建项目</h3>
<p>首先给项目创建一个仓库，不妨叫做<code>MyOS</code></p>
<p>之后，为项目创建一个编译用文件夹<code>build</code>，一个存放镜像文件的文件夹<code>run</code>，一个存放所有代码的文件夹<code>src</code></p>
<p>由于目前我们需要完成的是操作系统启动的部分，因此本节的代码<code>mbr.asm</code>放置在<code>src/boot</code>文件夹下，同时，为了能够方便地编译项目，在<code>build</code>文件夹下创建项目的<code>makefile</code>文件</p>
<p>创建完成之后的项目目录看起来就像这样</p>
<figure class="highlight plaintext"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line">.</span><br><span class="line">├── build</span><br><span class="line">│   └── makefile</span><br><span class="line">├── run</span><br><span class="line">└── src</span><br><span class="line">    └── boot</span><br><span class="line">        └── mbr.asm</span><br></pre></td></tr></table></figure>
<h3 id="Hello-World">Hello World</h3>
<p>Hello World实验作为各大语言的必经之路，在操统实验中自然也是必不可少的</p>
<p>在尝试输出代码之前，先研究一下怎样让操作系统显示出我们想要的东西…</p>
<h4 id="通过显存显示内容">通过显存显示内容</h4>
<p>qemu显示屏实际上是按25x80个字符来排列的矩阵，如下所示</p>
<p><img src="/img/os-journal-vol-2/IMG_20220719-114214235.png" alt="图 1"></p>
<p>为了便于控制显示的内容，IA-32处理器将这个矩阵映射到了内存地址的<code>0xB8000~0xBFFFF</code>处，这段地址被称为显存地址</p>
<p>在文本模式下，控制器的最小可控制单位为字符。每一个显示字符自上而下，从左到右依次使用显存中的两个字节表示，低字节表示显示的字符，高字节表示字符的颜色属性。<br>
例如，黑底白字的<code>H</code>在显存中如下存放：</p>
<table>
<thead>
<tr>
<th style="text-align:center">显存地址</th>
<th style="text-align:center">值</th>
<th style="text-align:center">含义</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">…</td>
<td style="text-align:center">-</td>
<td style="text-align:center">-</td>
</tr>
<tr>
<td style="text-align:center">B800:0001</td>
<td style="text-align:center">0x0F</td>
<td style="text-align:center">背景色黑色，前景色白色</td>
</tr>
<tr>
<td style="text-align:center">B800:0000</td>
<td style="text-align:center">0x48</td>
<td style="text-align:center">字符<code>H</code></td>
</tr>
</tbody>
</table>
<p>每个字符的颜色又由两部分组成，高4位为字符的背景色，低4位为字符的前景色，每个颜色由<code>R</code>、<code>G</code>、<code>B</code>和<code>K/I</code>位组成</p>
<p><code>K/I</code>位含义如下</p>
<table>
<thead>
<tr>
<th style="text-align:center">K/I</th>
<th style="text-align:center">背景色</th>
<th style="text-align:center">前景色</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">0</td>
<td style="text-align:center">不闪烁</td>
<td style="text-align:center">深色</td>
</tr>
<tr>
<td style="text-align:center">1</td>
<td style="text-align:center">闪烁</td>
<td style="text-align:center">亮（浅）色</td>
</tr>
</tbody>
</table>
<p>字符颜色的对照表如下</p>
<table>
<thead>
<tr>
<th style="text-align:center">R</th>
<th style="text-align:center">G</th>
<th style="text-align:center">B</th>
<th style="text-align:center">颜色</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">黑色</td>
</tr>
<tr>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">蓝色</td>
</tr>
<tr>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">绿色</td>
</tr>
<tr>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">1</td>
<td style="text-align:center">青色</td>
</tr>
<tr>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">红色</td>
</tr>
<tr>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">品红</td>
</tr>
<tr>
<td style="text-align:center">1</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">棕色</td>
</tr>
<tr>
<td style="text-align:center">1</td>
<td style="text-align:center">1</td>
<td style="text-align:center">1</td>
<td style="text-align:center">白色</td>
</tr>
</tbody>
</table>
<p>也就是说，<code>0x0F</code>对应的是 <strong>背景色为黑色不闪烁，前景色为亮白色的颜色</strong></p>
<p>于是，记 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mo stretchy="false">(</mo><mi>x</mi><mo separator="true">,</mo><mi>y</mi><mo stretchy="false">)</mo></mrow><annotation encoding="application/x-tex">(x, y)</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord mathnormal">x</span><span class="mpunct">,</span><span class="mspace" style="margin-right:0.1667em;"></span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span><span class="mclose">)</span></span></span></span> 为第 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mi>x</mi></mrow><annotation encoding="application/x-tex">x</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.4306em;"></span><span class="mord mathnormal">x</span></span></span></span> 行 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mi>y</mi></mrow><annotation encoding="application/x-tex">y</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.625em;vertical-align:-0.1944em;"></span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span></span></span></span> 列，如果想要在 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mo stretchy="false">(</mo><mi>x</mi><mo separator="true">,</mo><mi>y</mi><mo stretchy="false">)</mo></mrow><annotation encoding="application/x-tex">(x, y)</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord mathnormal">x</span><span class="mpunct">,</span><span class="mspace" style="margin-right:0.1667em;"></span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span><span class="mclose">)</span></span></span></span> 处显示一个字符，则需要做</p>
<ol>
<li>计算出该坐标相对于显存起始地址的偏移量</li>
<li>获得字符的颜色值</li>
<li>将字符ASCII码和颜色值放置在显存中</li>
</ol>
<p>其中，由于一行有80列，每个字符占用两个字节的空间，因此偏移地址计算的公式如下</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mi>I</mi><mi>n</mi><mi>d</mi><mi>e</mi><mi>x</mi><mo>=</mo><mn>2</mn><mo>×</mo><mo stretchy="false">(</mo><mi>x</mi><mo>×</mo><mn>80</mn><mo>+</mo><mi>y</mi><mo stretchy="false">)</mo></mrow><annotation encoding="application/x-tex">Index = 2 \times (x \times 80 + y)
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.6944em;"></span><span class="mord mathnormal" style="margin-right:0.07847em;">I</span><span class="mord mathnormal">n</span><span class="mord mathnormal">d</span><span class="mord mathnormal">e</span><span class="mord mathnormal">x</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.7278em;vertical-align:-0.0833em;"></span><span class="mord">2</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord mathnormal">x</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.7278em;vertical-align:-0.0833em;"></span><span class="mord">80</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span><span class="mclose">)</span></span></span></span></span></p>
<p>之所以要用偏移地址，是因为很容易发现，<code>0xB8000</code>这一个起始地址已经超出了16位所能表示的范围，因此需要引入段地址，这里再回顾一次引用段地址后实际地址的计算公式：</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mi>A</mi><mi>c</mi><mi>t</mi><mi>u</mi><mi>a</mi><mi>l</mi><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi><mi>e</mi><mi>s</mi><mi>s</mi><mo>=</mo><mo stretchy="false">(</mo><mi>S</mi><mi>e</mi><mi>g</mi><mi>m</mi><mi>e</mi><mi>n</mi><mi>t</mi><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi><mi>e</mi><mi>s</mi><mi>s</mi><mo>&lt;</mo><mo>&lt;</mo><mn>4</mn><mo stretchy="false">)</mo><mo>+</mo><mi>S</mi><mi>h</mi><mi>i</mi><mi>f</mi><mi>t</mi><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi><mi>e</mi><mi>s</mi><mi>s</mi></mrow><annotation encoding="application/x-tex">ActualAddress = (SegmentAddress &lt;&lt; 4) + ShiftAddress
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.6944em;"></span><span class="mord mathnormal">A</span><span class="mord mathnormal">c</span><span class="mord mathnormal">t</span><span class="mord mathnormal">u</span><span class="mord mathnormal">a</span><span class="mord mathnormal" style="margin-right:0.01968em;">l</span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal">ress</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">e</span><span class="mord mathnormal" style="margin-right:0.03588em;">g</span><span class="mord mathnormal">m</span><span class="mord mathnormal">e</span><span class="mord mathnormal">n</span><span class="mord mathnormal">t</span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal">ress</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">&lt;&lt;</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mord">4</span><span class="mclose">)</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">hi</span><span class="mord mathnormal" style="margin-right:0.10764em;">f</span><span class="mord mathnormal">t</span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal">ress</span></span></span></span></span></p>
<p>很显然，将段地址设置为<code>0xB800</code>可以让偏移地址从<code>0x0</code>开始取值，因此不妨将段地址<code>0xB800</code>放置在附加段寄存器<code>gs</code>中，然后使用<code>[gs:index]</code>格式进行取址，例如</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">gs</span>:<span class="number">0</span>], <span class="string">&#x27;H&#x27;</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">gs</span>:<span class="number">1</span>], <span class="number">0x0F</span></span><br></pre></td></tr></table></figure>
<p>就是将黑底白字的<code>H</code>显示在坐标 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mo stretchy="false">(</mo><mn>0</mn><mo separator="true">,</mo><mn>0</mn><mo stretchy="false">)</mo></mrow><annotation encoding="application/x-tex">(0, 0)</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord">0</span><span class="mpunct">,</span><span class="mspace" style="margin-right:0.1667em;"></span><span class="mord">0</span><span class="mclose">)</span></span></span></span>处</p>
<h4 id="通过中断显示内容">通过中断显示内容</h4>
<p>除了使用显存赋值这一操作来显示字符，在实模式下还提供了另一种显示字符的方式——中断</p>
<p>中断的调用涉及了四个部分：</p>
<ol>
<li><strong>设置中断参数</strong>：参数一般为存储在<code>bx</code>、<code>cx</code>和<code>dx</code>寄存器或是它们的高低位中的值</li>
<li><strong>设置中断功能</strong>：一种中断调用可以实现很多不同的功能，具体要调用的功能一般通过<code>ax</code>或是<code>ah</code>指定</li>
<li><strong>调用中断</strong>：使用<code>int</code>指令和对应的中断向量号调用中断</li>
<li><strong>获取返回值</strong>：中断函数如果有返回值的话一般是放在寄存器中，所以要从寄存器中获取返回值</li>
</ol>
<p>BIOS提供了很多中断函数，中断向量号和它们的功能可以查看 <a target="_blank" rel="noopener" href="https://wiki.osdev.org/BIOS">OSDev Wiki中关于BIOS的页面</a></p>
<p>在本次实验中我们暂时只关注<code>int 10h</code>这个函数，它提供了光标相关的操作，对应的参数和返回值可以参考 <a target="_blank" rel="noopener" href="https://zh.wikipedia.org/wiki/INT_10H">Wikipedia 中关于光标中断的页面</a><br>
为了方便查阅，显示字符所需要用到功能在下表列出</p>
<table>
<thead>
<tr>
<th style="text-align:center">功能</th>
<th style="text-align:center">功能号</th>
<th style="text-align:center">参数</th>
<th style="text-align:center">返回值</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">设置光标位置</td>
<td style="text-align:center"><code>AH=02H</code></td>
<td style="text-align:center"><code>BH</code>：页码<br><code>DH</code>：行，<code>DL</code>：列</td>
<td style="text-align:center">-</td>
</tr>
<tr>
<td style="text-align:center">获取光标位置</td>
<td style="text-align:center"><code>AH=03H</code></td>
<td style="text-align:center"><code>BX</code>：页码</td>
<td style="text-align:center"><code>AX=0</code><br><code>CH</code>：行扫描开始，<code>CL</code>：行扫描结束<br><code>DH</code>：行，<code>DL</code>：列<br><strong>实验中一般只需要用到<code>DH</code>和<code>DL</code></strong></td>
</tr>
<tr>
<td style="text-align:center">在光标位置写入字符</td>
<td style="text-align:center"><code>AH=09H</code></td>
<td style="text-align:center"><code>AL</code>：字符，<code>BL</code>：颜色<br><code>BH</code>：页码，<code>CX</code>：输出字符个数</td>
<td style="text-align:center">-</td>
</tr>
</tbody>
</table>
<p>所以在 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mo stretchy="false">(</mo><mi>x</mi><mo separator="true">,</mo><mi>y</mi><mo stretchy="false">)</mo></mrow><annotation encoding="application/x-tex">(x, y)</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord mathnormal">x</span><span class="mpunct">,</span><span class="mspace" style="margin-right:0.1667em;"></span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span><span class="mclose">)</span></span></span></span> 处写入字符需要如下步骤：</p>
<ol>
<li>设置光标位置为 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mi>x</mi></mrow><annotation encoding="application/x-tex">x</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.4306em;"></span><span class="mord mathnormal">x</span></span></span></span> 行 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mi>y</mi></mrow><annotation encoding="application/x-tex">y</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.625em;vertical-align:-0.1944em;"></span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span></span></span></span> 列</li>
<li>设置字符颜色等参数</li>
<li>在光标位置写入字符</li>
</ol>
<p>例如，在 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mo stretchy="false">(</mo><mn>0</mn><mi>x</mi><mn>1</mn><mi>C</mi><mo separator="true">,</mo><mn>0</mn><mi>x</mi><mn>03</mn><mo stretchy="false">)</mo></mrow><annotation encoding="application/x-tex">(0x1C, 0x03)</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord">0</span><span class="mord mathnormal">x</span><span class="mord">1</span><span class="mord mathnormal" style="margin-right:0.07153em;">C</span><span class="mpunct">,</span><span class="mspace" style="margin-right:0.1667em;"></span><span class="mord">0</span><span class="mord mathnormal">x</span><span class="mord">03</span><span class="mclose">)</span></span></span></span>处写入黑底白色的字符<code>H</code>操作为</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br></pre></td><td class="code"><pre><span class="line"><span class="symbol">move_cursor:</span></span><br><span class="line">  <span class="comment">; Set cordinates</span></span><br><span class="line">  <span class="keyword">mov</span> <span class="number">dh</span>, <span class="number">0x1C</span></span><br><span class="line">  <span class="keyword">mov</span> <span class="built_in">dl</span>, <span class="number">0x03</span></span><br><span class="line"></span><br><span class="line">  <span class="comment">; Set interrupt function</span></span><br><span class="line">  <span class="keyword">mov</span> <span class="number">ah</span>, <span class="number">0x02</span>  <span class="comment">; AH=02H indicates setting the position of cursor</span></span><br><span class="line"></span><br><span class="line">  <span class="comment">; Call interrupt function</span></span><br><span class="line">  <span class="keyword">int</span> <span class="number">0x10</span></span><br><span class="line"><span class="symbol"></span></span><br><span class="line"><span class="symbol">output_character:</span></span><br><span class="line">  <span class="comment">; Set character properties</span></span><br><span class="line">  <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="string">&#x27;H&#x27;</span>   <span class="comment">; Character: &#x27;H&#x27;</span></span><br><span class="line">  <span class="keyword">mov</span> <span class="built_in">bl</span>, <span class="number">0x0F</span>  <span class="comment">; Background: black &amp; Foreground: white</span></span><br><span class="line">  <span class="keyword">mov</span> <span class="built_in">cx</span>, <span class="number">0x01</span>  <span class="comment">; Output num: 1</span></span><br><span class="line"></span><br><span class="line">  <span class="comment">; Set interrupt function</span></span><br><span class="line">  <span class="keyword">mov</span> <span class="number">ah</span>, <span class="number">0x09</span>  <span class="comment">; AH=09H indicates outputting character at cursor position</span></span><br><span class="line"></span><br><span class="line">  <span class="comment">; Call interrupt function</span></span><br><span class="line">  <span class="keyword">int</span> <span class="number">0x10</span></span><br></pre></td></tr></table></figure>
<h4 id="编写MBR">编写MBR</h4>
<div class="note blue icon-padding flat"><i class="note-icon fas fa-history"></i><p><strong>回顾：MBR的加载</strong><br>
计算机加电启动并完成自检之后，BIOS会根据引导顺序检查磁盘首扇区 <em>（也即存放MBR的扇区）</em> 是否可以启动<br>
如果可以启动，BIOS会<strong>将这512字节加载到<code>0x7C00</code>处开始执行</strong><br>
此时，CPU运行在 <strong>实模式</strong> 下</p>
</div>
<p>为了让编译器能够正确理解 <em>“我们是在编写MBR”</em> 这件事，需要使用一些汇编的伪指令告知编译器：</p>
<ul>
<li>代码会被加载到<code>0x7C00</code>处执行</li>
<li>代码在16位模式下运行</li>
</ul>
<p>翻译成<code>asm</code>语言如下</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">[org <span class="number">0x7C00</span>]</span><br><span class="line">[<span class="meta">bits</span> <span class="number">16</span>]</span><br></pre></td></tr></table></figure>
<p>之后，需要对寄存器进行初始化，避免为初始化的段地址影响后续的取址操作<br>
需要注意的是，<strong>段寄存器只能通过<code>ax</code>寄存器赋值</strong>，所以可以先将<code>ax</code>置<code>0</code>，然后再通过<code>ax</code>将段寄存器置<code>0</code></p>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：不要初始化<code>cs</code></strong><br>
在初始化段寄存器的时候，通过实验发现，<code>ds</code>, <code>ss</code>, <code>es</code>, <code>fs</code>, <code>gs</code>都可以正常进行置<code>0</code>操作<br>
但是如果对<code>cs</code>进行置<code>0</code>，则会导致代码出现意料之外的行为<br>
所以不要将<code>cs</code>寄存器初始化</p>
</div>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Initialize registers</span></span><br><span class="line"><span class="keyword">xor</span> <span class="built_in">ax</span>, <span class="built_in">ax</span>  <span class="comment">; ax = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ds</span>, <span class="built_in">ax</span>  <span class="comment">; ds = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ss</span>, <span class="built_in">ax</span>  <span class="comment">; ss = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">es</span>, <span class="built_in">ax</span>  <span class="comment">; es = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">fs</span>, <span class="built_in">ax</span>  <span class="comment">; fs = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">gs</span>, <span class="built_in">ax</span>  <span class="comment">; gs = 0</span></span><br></pre></td></tr></table></figure>
<p>此时栈指针寄存器<code>sp</code>还没有被初始化，由于栈是从高地址向低地址增长的，所以需要给栈分配一段可以向下增长的空间<br>
这一段空间的选择相对自由，但是由于MBR对栈空间的使用较小，并且当启动到后续阶段的时候也可以再移动栈指针，所以不妨将栈指针放置在<code>0x7C00</code>处让其向下增长</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Initialize stack pointer</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">sp</span>, <span class="number">0x7C00</span></span><br></pre></td></tr></table></figure>
<p>之后使用显存显示内容的方式来完成<code>Hello World!</code>字串的输出</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Print something</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="number">0xB800</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">gs</span>, <span class="built_in">ax</span>  <span class="comment">; Set video segment as 0xB800</span></span><br><span class="line"></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">si</span>, <span class="number">0</span>   <span class="comment">; Current character index</span></span><br><span class="line"><span class="symbol">print:</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dl</span>, <span class="built_in">byte</span> [_msg + <span class="built_in">si</span>]    <span class="comment">; Fetch character</span></span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">dl</span>, <span class="number">0</span>                   <span class="comment">; Exit if character is &#x27;\0&#x27;</span></span><br><span class="line">    <span class="keyword">je</span>  print_exit</span><br><span class="line">    <span class="keyword">inc</span> <span class="built_in">si</span>                      <span class="comment">; index += 1</span></span><br><span class="line">    <span class="comment">; Calculate cordinate in vga memory</span></span><br><span class="line">    <span class="comment">; Shift = 2 * (80 * row  + col)</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [_row] </span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">ax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [_col]</span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">ax</span>, <span class="number">2</span></span><br><span class="line">    <span class="comment">; Copy character to memory</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">bx</span>, <span class="built_in">ax</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">gs</span>:<span class="built_in">bx</span>], <span class="built_in">dl</span>        <span class="comment">; Character to be printed</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">gs</span>:<span class="built_in">bx</span> + <span class="number">1</span>], <span class="number">0x0F</span>  <span class="comment">; Black background with white foreground</span></span><br><span class="line">    <span class="comment">; Set new cordinate</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">word</span> [_col], <span class="number">1</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [_col]</span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">ax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">jne</span> add_row_exit</span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">word</span> [_row], <span class="number">1</span>          <span class="comment">; move cursor to next row if col == 80</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">word</span> [_col], <span class="number">0</span></span><br><span class="line"><span class="symbol">    add_row_exit:</span></span><br><span class="line">    <span class="comment">; Reture to top</span></span><br><span class="line">    <span class="keyword">jmp</span> print</span><br><span class="line"><span class="symbol">print_exit:</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Halt here</span></span><br><span class="line"><span class="keyword">jmp</span> $</span><br><span class="line"></span><br><span class="line"><span class="comment">; Variables</span></span><br><span class="line">_row    <span class="built_in">dw</span> <span class="number">0</span></span><br><span class="line">_col    <span class="built_in">dw</span> <span class="number">0</span></span><br><span class="line">_msg    <span class="built_in">db</span> <span class="string">&#x27;Hello World!&#x27;</span>,</span><br><span class="line">        <span class="built_in">db</span> <span class="number">0</span></span><br></pre></td></tr></table></figure>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：不要将数据放置在代码中间或者前部</strong><br>
BIOS从<code>0x7C00</code>处开始执行代码，其并不区分内存中存储的二进制码究竟是数据还是指令，因此如果将数据放置在代码前面，可能会让CPU误认为数据是代码从而被执行，产生无法预料的错误，这类错误往往很难通过调试发现</p>
</div>
<div class="note info flat"><p><strong>不要让代码掉进 <em>“虚无”</em></strong><br>
可以在代码后面添加<code>jmp $</code>，其中<code>$</code>的意思是当前语句的地址，这句指令无限跳转到其自身执行<br>
这阻止了CPU继续执行这句代码之后的内容，因为后面可能是数据空间或是虚无，它们同样可能被解读成错误的指令从而产生无法预料的错误</p>
</div>
<p>最后，由于还没有开始做文件系统和磁盘分区，不妨将MBR后续字节填充零，使用伪指令<code>times</code>来重复填充，<code>$ - $$</code>很容易解读出其意译为当前地址减去文件起始地址的差</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Fill 0 before byte 447</span></span><br><span class="line"><span class="built_in">times</span> <span class="number">446</span> - ($ - $$) <span class="built_in">db</span> <span class="number">0</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Fill 0 (temporary) for partition table</span></span><br><span class="line"><span class="built_in">times</span> <span class="number">510</span> - ($ - $$) <span class="built_in">db</span> <span class="number">0</span></span><br></pre></td></tr></table></figure>
<div class="note success flat"><p><strong>完整的<code>mbr.asm</code>文件如下</strong></p>
</div>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br><span class="line">44</span><br><span class="line">45</span><br><span class="line">46</span><br><span class="line">47</span><br><span class="line">48</span><br><span class="line">49</span><br><span class="line">50</span><br><span class="line">51</span><br><span class="line">52</span><br><span class="line">53</span><br><span class="line">54</span><br><span class="line">55</span><br><span class="line">56</span><br><span class="line">57</span><br><span class="line">58</span><br><span class="line">59</span><br><span class="line">60</span><br><span class="line">61</span><br><span class="line">62</span><br></pre></td><td class="code"><pre><span class="line">[org <span class="number">0x7C00</span>]  <span class="comment">; Indicates codes below starts at 0x7C00</span></span><br><span class="line">[<span class="meta">bits</span> <span class="number">16</span>]   <span class="comment">; Indicates codes below run in 16-bit mode</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Initialize registers</span></span><br><span class="line"><span class="keyword">xor</span> <span class="built_in">ax</span>, <span class="built_in">ax</span>  <span class="comment">; ax = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ds</span>, <span class="built_in">ax</span>  <span class="comment">; ds = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ss</span>, <span class="built_in">ax</span>  <span class="comment">; ss = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">es</span>, <span class="built_in">ax</span>  <span class="comment">; es = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">fs</span>, <span class="built_in">ax</span>  <span class="comment">; fs = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">gs</span>, <span class="built_in">ax</span>  <span class="comment">; gs = 0</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Initialize stack pointer</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">sp</span>, <span class="number">0x7C00</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Print something</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="number">0xB800</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">gs</span>, <span class="built_in">ax</span>  <span class="comment">; Set video segment</span></span><br><span class="line"></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">si</span>, <span class="number">0</span></span><br><span class="line"><span class="symbol">print:</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dl</span>, <span class="built_in">byte</span> [_msg + <span class="built_in">si</span>]</span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">dl</span>, <span class="number">0</span></span><br><span class="line">    <span class="keyword">je</span>  print_exit</span><br><span class="line">    <span class="keyword">inc</span> <span class="built_in">si</span></span><br><span class="line">    <span class="comment">; Calculate cordinate in vga memory</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [_row]</span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">ax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [_col]</span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">ax</span>, <span class="number">2</span></span><br><span class="line">    <span class="comment">; Copy character to memory</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">bx</span>, <span class="built_in">ax</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">gs</span>:<span class="built_in">bx</span>], <span class="built_in">dl</span>    <span class="comment">; Character to be printed</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">gs</span>:<span class="built_in">bx</span> + <span class="number">1</span>], <span class="number">0x0F</span>  <span class="comment">; Black background with white foreground</span></span><br><span class="line">    <span class="comment">; Set new cordinate</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">word</span> [_col], <span class="number">1</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [_col]</span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">ax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">jne</span> add_row_exit</span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">word</span> [_row], <span class="number">1</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">word</span> [_col], <span class="number">0</span></span><br><span class="line"><span class="symbol">    add_row_exit:</span></span><br><span class="line">    <span class="comment">; Reture to top</span></span><br><span class="line">    <span class="keyword">jmp</span> print</span><br><span class="line"><span class="symbol">print_exit:</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Halt here</span></span><br><span class="line"><span class="keyword">jmp</span> $</span><br><span class="line"></span><br><span class="line"><span class="comment">; Variables</span></span><br><span class="line">_row    <span class="built_in">dw</span> <span class="number">0</span></span><br><span class="line">_col    <span class="built_in">dw</span> <span class="number">0</span></span><br><span class="line">_msg    <span class="built_in">db</span> <span class="string">&#x27;Hello World!&#x27;</span>,</span><br><span class="line">        <span class="built_in">db</span> <span class="number">0</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Fill 0 before byte 447</span></span><br><span class="line"><span class="built_in">times</span> <span class="number">446</span> - ($ - $$) <span class="built_in">db</span> <span class="number">0</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Fill 0 (temporary) for partition table</span></span><br><span class="line"><span class="built_in">times</span> <span class="number">510</span> - ($ - $$) <span class="built_in">db</span> <span class="number">0</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Magic number</span></span><br><span class="line"><span class="built_in">dw</span> <span class="number">0xAA55</span></span><br></pre></td></tr></table></figure>
<h4 id="编写makefile">编写makefile</h4>
<div class="note blue icon-padding flat"><i class="note-icon fas fa-bookmark"></i><p><strong>Makefile Tutorial</strong><br>
如果对Makefile语法尚不熟悉，可以前往 <a target="_blank" rel="noopener" href="https://makefiletutorial.com/">Makefile Tutorial</a> 页面了解</p>
</div>
<p>为了让项目能够顺利地运行起来，需要编写Makefile文件。在默认已经了解Makefile语法的前提下，简要介绍从MBR启动所需要的各个文件以及它们的依赖关系</p>
<table>
<thead>
<tr>
<th style="text-align:center">文件</th>
<th style="text-align:center">作用</th>
<th style="text-align:center">依赖关系</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center"><code>hd.img</code></td>
<td style="text-align:center">硬盘文件<br>QEMU会尝试从这个文件的首扇区加载MBR启动</td>
<td style="text-align:center"><code>mbr.bin</code><br>需要将MBR写入磁盘才能够启动</td>
</tr>
<tr>
<td style="text-align:center"><code>mbr.bin</code></td>
<td style="text-align:center">MBR编译后的二进制文件</td>
<td style="text-align:center"><code>mbr.asm</code></td>
</tr>
<tr>
<td style="text-align:center"><code>mbr.asm</code></td>
<td style="text-align:center">MBR源文件</td>
<td style="text-align:center">-</td>
</tr>
</tbody>
</table>
<p>根据这些依赖关系，可以写出Makefile文件</p>
<figure class="highlight makefile"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br></pre></td><td class="code"><pre><span class="line">ASM_COMPILER := nasm</span><br><span class="line"></span><br><span class="line">SRC_PATH := ../src</span><br><span class="line">RUN_PATH := ../run</span><br><span class="line"></span><br><span class="line"><span class="meta"><span class="keyword">.PHONY</span>:</span></span><br><span class="line"><span class="section">build: mbr.bin</span></span><br><span class="line">  qemu-img create <span class="variable">$(RUN_PATH)</span>/hd.img 10m</span><br><span class="line">  dd if=mbr.bin of=<span class="variable">$(RUN_PATH)</span>/hd.img bs=512 count=1 seek=0 conv=notrunc</span><br><span class="line"></span><br><span class="line"><span class="section">%.bin: <span class="variable">$(SRC_PATH)</span>/boot/%.asm</span></span><br><span class="line">  <span class="variable">$(ASM_COMPILER)</span> -o <span class="variable">$@</span> -f bin <span class="variable">$^</span></span><br><span class="line"></span><br><span class="line"><span class="meta"><span class="keyword">.PHONY</span>:</span></span><br><span class="line"><span class="section">clean:</span></span><br><span class="line">  rm -f *.o* *.bin</span><br><span class="line">  rm -f <span class="variable">$(RUN_PATH)</span>/*.img</span><br><span class="line"></span><br><span class="line"><span class="meta"><span class="keyword">.PHONY</span>:</span></span><br><span class="line"><span class="section">run:</span></span><br><span class="line">  qemu-system-i386 -hda <span class="variable">$(RUN_PATH)</span>/hd.img -vga virtio -serial null -parallel stdio -no-reboot</span><br></pre></td></tr></table></figure>
<h4 id="运行和结果">运行和结果</h4>
<p>进入项目目录下的<code>build/</code>目录</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">make clean build run</span><br></pre></td></tr></table></figure>
<p>可以首先清除原先生成的文件，重新构建并且运行<br>
运行的结果如下</p>
<p><img src="/img/os-journal-vol-2/IMG_20220719-155923542.png" alt="图 2"></p>
<div class="note success flat"><p><strong>完成本章</strong><br>
下一章中，将会编写bootloader，从mbr中加载bootloader并且启动，最后在bootloader中让CPU进入保护模式<br>
如果准备好了的话，就让我们进入<a href="https://blog.linloir.cn/2022/07/19/os-journal-vol-3/">下一章</a>吧！</p>
</div>

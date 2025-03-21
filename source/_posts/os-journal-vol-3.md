---
title: 操统实验日志 第三章 从实模式到保护模式
date: 2022-07-19 13:36:30
description: 在本章的第一部分中，将会介绍读取硬盘的CHS方式、LBA方式以及如何通过 in、out 指令读写硬盘，之后会将上一章输出 Hello World! 的代码移植到 BootLoader 中，并且从 MBR 中加载并跳转到编写的 BootLoader 执行。在第二部分中，会回顾保护模式的概念并介绍进入保护模式的四个步骤，并在开启保护模式之后输出第二个 Hello World。
tags:
  - 操作系统
  - 大学
categories:
  - 技术
---

<h2 id="关于本章">关于本章</h2>
<p>在本章的<a href="#%E7%AC%AC%E4%B8%80%E6%AC%A1%E8%B7%83%E8%BF%9B%E4%BB%8Embr%E8%B7%B3%E8%BD%AC%E5%88%B0bootloader">第一部分</a>中，将会介绍读取硬盘的CHS方式、LBA方式以及如何通过<code>in</code>、<code>out</code>指令读写硬盘，之后会将<a href="https://blog.linloir.cn/2022/07/16/os-journal-vol-2/index.html#%E7%BC%96%E5%86%99MBR">上一章</a>输出<code>Hello World!</code>的代码移植到BootLoader中，并且从MBR中加载并跳转到编写的BootLoader执行</p>
<p>在<a href="#%E5%9C%A8bootloader%E4%B8%AD%E5%BC%80%E5%90%AF%E4%BF%9D%E6%8A%A4%E6%A8%A1%E5%BC%8F">第二部分</a>中，会回顾保护模式的概念并介绍进入保护模式的四个步骤，并在开启保护模式之后输出第二个<code>Hello World</code></p>
<h2 id="第一次跃进：从MBR跳转到BootLoader">第一次跃进：从MBR跳转到BootLoader</h2>
<p>MBR只有512字节的大小，甚至如果除去分区表，只有四百多字节的大小，这个大小稍微复杂一些的程序就难以运行了，更不要说是把全部的操作系统都放在里面了</p>
<p>因此，目前的首要任务就是 <strong>突破512字节的限制</strong>，从而能够让我们着手编写更复杂的程序，其实，聪明的工程师很早就提出了一个可行的解决方案：<strong>在MBR里面把另一段更大的程序加载到内存里面，之后跳转到加载的地址去执行</strong>，这个 <em>“更大的程序”</em> 就是我们本节要编写的BootLoader，它在编译之后会被烧录到硬盘镜像中指定的扇区里，MBR要做的事情就是从这些扇区里面把我们编写的BootLoader加载出来放在内存里，然后跳转到加载的内存地址处执行BootLoader的代码</p>
<div class="note orange icon-padding flat"><i class="note-icon fas fa-question-circle"></i><p><strong>为什么不直接从MBR跳转到内核运行？</strong><br>
从<a href="https://blog.linloir.cn/2022/07/16/os-journal-vol-2/index.html#%E5%AE%9E%E9%AA%8C%E8%B7%AF%E7%BA%BF">上一章的实验路线</a>中可以看到，在加载内核之前有许多需要准备的工作，包括开启虚拟分页机制以及文件系统的装载<br>
很显然这些并不能在MBR中完成，所以需要在MBR和内核中间添加一些步骤，作为启动内核前的准备操作</p>
</div>
<h3 id="简述CHS模式">简述CHS模式</h3>
<p>CHS全称为 <strong>Cylinder-head-sector</strong>，是早期用来定位硬盘上扇区的一种方式</p>
<p>在CHS中，扇区通过不同的 <strong>柱面 <em>(Cylinder)</em></strong>、<strong>磁头 <em>(Head)</em></strong> 和 <strong>扇区 <em>(Sector)</em></strong> 指定，除此之外，在CHS中还有一个额外的概念 <strong>磁道 <em>(Track)</em></strong> ，下面简要介绍这四个概念的含义</p>
<ul>
<li>
<p><strong>柱面</strong>：<br>
磁盘是由很多层碟片堆叠而成的，可以想象为一个实心的圆柱体。如果用一个与磁盘同心的空心圆柱去切割 <em>（或者说做相交的操作）</em>，则每一个磁盘都会被相交得到一个空心圆，这些空心圆的组合就叫做柱面，实际上，它们看起来也确实像一个圆柱的外表面，而这个圆柱与做相交时的空心圆柱大小是一样的</p>
<p>换一种说法，每一个碟片都是由很多个同心圆组成的，这些同心圆被叫做 <strong>磁道</strong>，从外到内由0开始编号，每个碟片上的磁道数都是相同而且对齐的，也就是说，如果我们俯视一个磁盘，那么最上面的碟片的磁道是怎么分布的，它下面的碟片也都是这么分布的，如果我们对所有碟片选定同一个磁道，那么它从上到下看起来就会构成一个圆柱的外表面，也就是 <strong>柱面</strong>，这个磁道编号就是柱面号</p>
</li>
<li>
<p><strong>磁头</strong>：<br>
一个磁盘有很多个碟片，对应地，每个碟片也有一个磁头用来读写数据，磁头从上往下由0开始编号，这个编号就是磁头号。一般来说，磁头会停留在一个同心圆 <em>（也就是磁道）</em> 上，并且可以在不同的磁道间移动</p>
</li>
<li>
<p><strong>扇区</strong><br>
磁盘的每一个扇区都可以按照旋转角度被等分为很多段圆弧，每一段圆弧就被称作一个扇区，扇区大小一般为512字节</p>
<div class="note info flat"><p><strong>扇区与数据密度</strong><br>
很容易发现，如果对整个碟片上所有的磁道都按照一个单一的旋转角度来划分扇区，则每个扇区虽然对应的圆心角相同、数据容量相同，但是长度却不一样，这就会让外圈的磁道数据密度降低<br>
事实上，<strong>老式的磁盘使用的就是这种方式</strong>，确实会导致内外磁道数据密度不同的问题<br>
而目前有<strong>新的解决方式按照等密度的方式划分扇区，也就是越靠外的磁道扇区数越多</strong></p>
<p>在实验中将使用经典的理解，也就是老式磁盘的划分方式，同时后面也会看到在LBA模式中这种差异将不会对取址产生影响</p>
</div>
</li>
<li>
<p><strong>磁道</strong>：<br>
每一个碟片都是由很多个同心圆组成的，这些同心圆被叫做 <strong>磁道</strong>。同时，磁道也可以理解为是由柱面和磁头对应的盘面相交得到的圆</p>
<p>也就是说，一个柱面和一个磁头可以唯一指定一个磁道，在CHS取址模式中不需要提供磁道号也正是这个原因</p>
</li>
</ul>
<p>如果还不能够完全理解，可以查看<a target="_blank" rel="noopener" href="https://en.wikipedia.org/wiki/Cylinder-head-sector">Wikipedia关于CHS模式的页面</a>中精妙的配图如下</p>
<p><img src="/img/os-journal-vol-3/IMG_20220721-161302861.png" alt="图 1"></p>
<h3 id="简述LBA模式">简述LBA模式</h3>
<p>LBA模式全称为 <strong>Logical Block Addressing</strong>，顾名思义，就是通过逻辑扇区号去寻址物理扇区号，具体的寻址方式则是对用户透明的</p>
<p>在LBA模式下，不存在像CHS那样复杂的概念，<strong>硬盘的扇区被看作是线性的并从0扇区开始编号</strong>，当使用LBA扇区号进行取址的时候，存储设备会自动转换成对应的柱面、磁头和扇区进行读取，用户不需要关心这部分的实现</p>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：CHS和LBA模式的扇区编号不同</strong><br>
在CHS模式下，扇区从1开始编号，而在LBA模式下，扇区从0开始编号</p>
</div>
<p>通过CHS编号来计算LBA编号可以使用如下公式，其中<code>HPC</code>为柱面中的磁头数 <em>Heads per Cylinder</em>，<code>SPT</code>为每个磁道中的扇区数 <em>Sectors per Track</em></p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mi>L</mi><mi>B</mi><mi>A</mi><mo>=</mo><mo stretchy="false">(</mo><mi>C</mi><mo>×</mo><mi>H</mi><mi>P</mi><mi>C</mi><mo>+</mo><mi>H</mi><mo stretchy="false">)</mo><mo>×</mo><mi>S</mi><mi>P</mi><mi>T</mi><mo>+</mo><mo stretchy="false">(</mo><mi>S</mi><mo>−</mo><mn>1</mn><mo stretchy="false">)</mo></mrow><annotation encoding="application/x-tex">LBA = (C \times HPC + H) \times SPT + (S - 1)
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.6833em;"></span><span class="mord mathnormal">L</span><span class="mord mathnormal" style="margin-right:0.05017em;">B</span><span class="mord mathnormal">A</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord mathnormal" style="margin-right:0.07153em;">C</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal" style="margin-right:0.08125em;">H</span><span class="mord mathnormal" style="margin-right:0.07153em;">PC</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mord mathnormal" style="margin-right:0.08125em;">H</span><span class="mclose">)</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal" style="margin-right:0.13889em;">SPT</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mopen">(</span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">−</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mord">1</span><span class="mclose">)</span></span></span></span></span></p>
<p>同样，LBA模式也可以通过下述公式转换为CHS编号</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mo fence="true">{</mo><mtable rowspacing="0.25em" columnalign="right left" columnspacing="0em"><mtr><mtd><mstyle scriptlevel="0" displaystyle="true"><mtext>  </mtext></mstyle></mtd><mtd><mstyle scriptlevel="0" displaystyle="true"><mrow><mrow></mrow><mi>C</mi><mo>=</mo><mi>L</mi><mi>B</mi><mi>A</mi><mo>÷</mo><mo stretchy="false">(</mo><mi>H</mi><mi>P</mi><mi>C</mi><mo>×</mo><mi>S</mi><mi>P</mi><mi>T</mi><mo stretchy="false">)</mo></mrow></mstyle></mtd></mtr><mtr><mtd><mstyle scriptlevel="0" displaystyle="true"><mrow></mrow></mstyle></mtd><mtd><mstyle scriptlevel="0" displaystyle="true"><mrow><mrow></mrow><mi>H</mi><mo>=</mo><mo stretchy="false">(</mo><mi>L</mi><mi>B</mi><mi>A</mi><mo>÷</mo><mi>S</mi><mi>P</mi><mi>T</mi><mo stretchy="false">)</mo><mtext>  </mtext><mi>m</mi><mi>o</mi><mi>d</mi><mtext>  </mtext><mi>H</mi><mi>P</mi><mi>C</mi></mrow></mstyle></mtd></mtr><mtr><mtd><mstyle scriptlevel="0" displaystyle="true"><mrow></mrow></mstyle></mtd><mtd><mstyle scriptlevel="0" displaystyle="true"><mrow><mrow></mrow><mi>S</mi><mo>=</mo><mo stretchy="false">(</mo><mi>L</mi><mi>B</mi><mi>A</mi><mtext>  </mtext><mi>m</mi><mi>o</mi><mi>d</mi><mtext>  </mtext><mi>S</mi><mi>P</mi><mi>T</mi><mo stretchy="false">)</mo><mo>+</mo><mn>1</mn></mrow></mstyle></mtd></mtr></mtable></mrow><annotation encoding="application/x-tex">\left\{
  \begin{aligned}
    \;  &amp;C = LBA \div (HPC \times SPT)\\
        &amp;H = (LBA \div SPT)\;mod\;HPC\\
        &amp;S = (LBA\;mod\;SPT) + 1
  \end{aligned}
\right.
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:4.5em;vertical-align:-2em;"></span><span class="minner"><span class="mopen"><span class="delimsizing mult"><span class="vlist-t vlist-t2"><span class="vlist-r"><span class="vlist" style="height:2.35em;"><span style="top:-2.2em;"><span class="pstrut" style="height:3.15em;"></span><span class="delimsizinginner delim-size4"><span>⎩</span></span></span><span style="top:-2.192em;"><span class="pstrut" style="height:3.15em;"></span><span style="height:0.316em;width:0.8889em;"><svg xmlns="http://www.w3.org/2000/svg" width='0.8889em' height='0.316em' style='width:0.8889em' viewBox='0 0 888.89 316' preserveAspectRatio='xMinYMin'><path d='M384 0 H504 V316 H384z M384 0 H504 V316 H384z'/></svg></span></span><span style="top:-3.15em;"><span class="pstrut" style="height:3.15em;"></span><span class="delimsizinginner delim-size4"><span>⎨</span></span></span><span style="top:-4.292em;"><span class="pstrut" style="height:3.15em;"></span><span style="height:0.316em;width:0.8889em;"><svg xmlns="http://www.w3.org/2000/svg" width='0.8889em' height='0.316em' style='width:0.8889em' viewBox='0 0 888.89 316' preserveAspectRatio='xMinYMin'><path d='M384 0 H504 V316 H384z M384 0 H504 V316 H384z'/></svg></span></span><span style="top:-4.6em;"><span class="pstrut" style="height:3.15em;"></span><span class="delimsizinginner delim-size4"><span>⎧</span></span></span></span><span class="vlist-s">​</span></span><span class="vlist-r"><span class="vlist" style="height:1.85em;"><span></span></span></span></span></span></span><span class="mord"><span class="mtable"><span class="col-align-r"><span class="vlist-t vlist-t2"><span class="vlist-r"><span class="vlist" style="height:2.5em;"><span style="top:-4.5em;"><span class="pstrut" style="height:2.84em;"></span><span class="mord"><span class="mspace" style="margin-right:0.2778em;"></span></span></span><span style="top:-3em;"><span class="pstrut" style="height:2.84em;"></span><span class="mord"></span></span><span style="top:-1.5em;"><span class="pstrut" style="height:2.84em;"></span><span class="mord"></span></span></span><span class="vlist-s">​</span></span><span class="vlist-r"><span class="vlist" style="height:2em;"><span></span></span></span></span></span><span class="col-align-l"><span class="vlist-t vlist-t2"><span class="vlist-r"><span class="vlist" style="height:2.5em;"><span style="top:-4.66em;"><span class="pstrut" style="height:3em;"></span><span class="mord"><span class="mord"></span><span class="mord mathnormal" style="margin-right:0.07153em;">C</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord mathnormal">L</span><span class="mord mathnormal" style="margin-right:0.05017em;">B</span><span class="mord mathnormal">A</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">÷</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mopen">(</span><span class="mord mathnormal" style="margin-right:0.08125em;">H</span><span class="mord mathnormal" style="margin-right:0.07153em;">PC</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mord mathnormal" style="margin-right:0.13889em;">SPT</span><span class="mclose">)</span></span></span><span style="top:-3.16em;"><span class="pstrut" style="height:3em;"></span><span class="mord"><span class="mord"></span><span class="mord mathnormal" style="margin-right:0.08125em;">H</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mopen">(</span><span class="mord mathnormal">L</span><span class="mord mathnormal" style="margin-right:0.05017em;">B</span><span class="mord mathnormal">A</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">÷</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mord mathnormal" style="margin-right:0.13889em;">SPT</span><span class="mclose">)</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord mathnormal">m</span><span class="mord mathnormal">o</span><span class="mord mathnormal">d</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord mathnormal" style="margin-right:0.08125em;">H</span><span class="mord mathnormal" style="margin-right:0.07153em;">PC</span></span></span><span style="top:-1.66em;"><span class="pstrut" style="height:3em;"></span><span class="mord"><span class="mord"></span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mopen">(</span><span class="mord mathnormal">L</span><span class="mord mathnormal" style="margin-right:0.05017em;">B</span><span class="mord mathnormal">A</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord mathnormal">m</span><span class="mord mathnormal">o</span><span class="mord mathnormal">d</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord mathnormal" style="margin-right:0.13889em;">SPT</span><span class="mclose">)</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mord">1</span></span></span></span><span class="vlist-s">​</span></span><span class="vlist-r"><span class="vlist" style="height:2em;"><span></span></span></span></span></span></span></span><span class="mclose nulldelimiter"></span></span></span></span></span></span></p>
<h3 id="在LBA模式下通过端口读写磁盘">在LBA模式下通过端口读写磁盘</h3>
<div class="note info flat"><p><strong>接口和模式</strong><br>
在实验中使用的接口为ATA，使用PIO模式进行访问</p>
</div>
<div class="note blue icon-padding flat"><i class="note-icon fas fa-book-open"></i><p><strong>ATA PIO模式</strong><br>
ATA PIO mode的更多信息可以查看<a target="_blank" rel="noopener" href="https://wiki.osdev.org/ATA_PIO_Mode">OSDev中关于这一主题的页面</a></p>
</div>
<div class="note blue icon-padding flat"><i class="note-icon fas fa-external-link-square-alt"></i><p><strong>ATA、PATA、SATA的区别</strong><br>
关于ATA、PATA、SATA以及ATA PIO之间的区别，可以查看<a target="_blank" rel="noopener" href="https://www.reddit.com/r/osdev/comments/mvzjdg/comment/gvfb8rq/?utm_source=share&amp;utm_medium=web2x&amp;context=3">Reddit中关于这个主题的讨论</a></p>
</div>
<p>了解了硬盘的寻址模式之后，就可以开始着手从磁盘里读取数据了，在实验中我就采用了更为 <s>无脑</s> 简单的LBA28模式</p>
<div class="note info flat"><p><strong>LBA28</strong><br>
LBA28指的是使用28位来表示逻辑扇区的编号</p>
</div>
<p>既然要从磁盘读取数据，那就自然要告诉磁盘 <strong>读数据、去哪读和读多少</strong>，端口就在这个过程中充当了信使的角色，我们通过向端口发送数据，告诉磁盘读取地址和读取数量，等候磁盘完成后再从端口接收数据</p>
<p>在磁盘读的操作中，要用到的端口及其描述如下</p>
<table>
<thead>
<tr>
<th style="text-align:center">端口号</th>
<th style="text-align:center">端口数据传输方向</th>
<th style="text-align:center">作用 (LBA28)</th>
<th style="text-align:center">描述</th>
<th style="text-align:center">位长 (LBA28)</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">0x1F0</td>
<td style="text-align:center">读/写</td>
<td style="text-align:center">数据寄存器</td>
<td style="text-align:center">硬盘读出/要写入的数据</td>
<td style="text-align:center">16-bit</td>
</tr>
<tr>
<td style="text-align:center">0x1F1</td>
<td style="text-align:center">读</td>
<td style="text-align:center">错误码寄存器</td>
<td style="text-align:center">存储执行的ATA指令所产生的错误码</td>
<td style="text-align:center">8-bit</td>
</tr>
<tr>
<td style="text-align:center">0x1F1</td>
<td style="text-align:center">写</td>
<td style="text-align:center">功能寄存器</td>
<td style="text-align:center">用来指定特定指令的接口功能</td>
<td style="text-align:center">8-bit</td>
</tr>
<tr>
<td style="text-align:center">0x1F2</td>
<td style="text-align:center">读/写</td>
<td style="text-align:center">扇区数寄存器</td>
<td style="text-align:center">存放需要读写的扇区数量</td>
<td style="text-align:center">8-bit</td>
</tr>
<tr>
<td style="text-align:center">0x1F3</td>
<td style="text-align:center">读/写</td>
<td style="text-align:center">起始扇区寄存器</td>
<td style="text-align:center">存放起始扇区0-7位</td>
<td style="text-align:center">8-bit</td>
</tr>
<tr>
<td style="text-align:center">0x1F4</td>
<td style="text-align:center">读/写</td>
<td style="text-align:center">起始扇区寄存器</td>
<td style="text-align:center">存放起始扇区8-15位</td>
<td style="text-align:center">8-bit</td>
</tr>
<tr>
<td style="text-align:center">0x1F5</td>
<td style="text-align:center">读/写</td>
<td style="text-align:center">起始扇区寄存器</td>
<td style="text-align:center">存放起始扇区16-23位</td>
<td style="text-align:center">8-bit</td>
</tr>
<tr>
<td style="text-align:center">0x1F6</td>
<td style="text-align:center">读/写</td>
<td style="text-align:center">磁盘、起始扇区寄存器</td>
<td style="text-align:center">选择磁盘和访问模式<br>存放起始扇区24-27位</td>
<td style="text-align:center">8-bit</td>
</tr>
<tr>
<td style="text-align:center">0x1F7</td>
<td style="text-align:center">读</td>
<td style="text-align:center">状态寄存器</td>
<td style="text-align:center">读取当前磁盘状态</td>
<td style="text-align:center">8-bit</td>
</tr>
<tr>
<td style="text-align:center">0x1F7</td>
<td style="text-align:center">写</td>
<td style="text-align:center">指令寄存器</td>
<td style="text-align:center">传送ATA指令</td>
<td style="text-align:center">8-bit</td>
</tr>
</tbody>
</table>
<p>其中，部分端口的位作用比较复杂，部分位在实验中也不需要留意，使用*号注明</p>
<table>
<thead>
<tr>
<th style="text-align:center">端口号</th>
<th style="text-align:center">位</th>
<th style="text-align:center">缩写</th>
<th style="text-align:center">作用</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">0x1F6</td>
<td style="text-align:center">0-3</td>
<td style="text-align:center">-</td>
<td style="text-align:center">在LBA模式中，指定其起始扇区号的24-27位</td>
</tr>
<tr>
<td style="text-align:center">0x1F6</td>
<td style="text-align:center">4</td>
<td style="text-align:center">DRV</td>
<td style="text-align:center">指定磁盘<br><code>0</code>：主硬盘<br><code>1</code>：从硬盘</td>
</tr>
<tr>
<td style="text-align:center">0x1F6</td>
<td style="text-align:center">5</td>
<td style="text-align:center">1</td>
<td style="text-align:center">始终置位</td>
</tr>
<tr>
<td style="text-align:center">0x1F6</td>
<td style="text-align:center">6</td>
<td style="text-align:center">LBA</td>
<td style="text-align:center">指定访问模式<br><code>0</code>：CHS模式<br><code>1</code>：LBA模式</td>
</tr>
<tr>
<td style="text-align:center">0x1F6</td>
<td style="text-align:center">7</td>
<td style="text-align:center">1</td>
<td style="text-align:center">始终置位</td>
</tr>
</tbody>
</table>
<table>
<thead>
<tr>
<th style="text-align:center">端口号</th>
<th style="text-align:center">位</th>
<th style="text-align:center">缩写</th>
<th style="text-align:center">作用</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">0x1F7 (Read)</td>
<td style="text-align:center">0</td>
<td style="text-align:center">ERR</td>
<td style="text-align:center">指示是否有错误发生，通过发送新指令可以清除该位</td>
</tr>
<tr>
<td style="text-align:center">0x1F7 (Read)</td>
<td style="text-align:center">1*</td>
<td style="text-align:center">IDX</td>
<td style="text-align:center">索引，始终置为0</td>
</tr>
<tr>
<td style="text-align:center">0x1F7 (Read)</td>
<td style="text-align:center">2*</td>
<td style="text-align:center">CORR</td>
<td style="text-align:center">修正数据，始终置位0</td>
</tr>
<tr>
<td style="text-align:center">0x1F7 (Read)</td>
<td style="text-align:center">3</td>
<td style="text-align:center">DRQ</td>
<td style="text-align:center"><code>0</code>：硬盘还不能交换数据<br><code>1</code>：硬盘存在可以读取的数据或是可以写入数据</td>
</tr>
<tr>
<td style="text-align:center">0x1F7 (Read)</td>
<td style="text-align:center">4*</td>
<td style="text-align:center">SRV</td>
<td style="text-align:center">重叠模式服务请求</td>
</tr>
<tr>
<td style="text-align:center">0x1F7 (Read)</td>
<td style="text-align:center">5*</td>
<td style="text-align:center">DF</td>
<td style="text-align:center">驱动器故障错误</td>
</tr>
<tr>
<td style="text-align:center">0x1F7 (Read)</td>
<td style="text-align:center">6*</td>
<td style="text-align:center">RDY</td>
<td style="text-align:center"><code>0</code>：驱动器发生了减速或是错误<br><code>1</code>：驱动器运转正常</td>
</tr>
<tr>
<td style="text-align:center">0x1F7 (Read)</td>
<td style="text-align:center">7</td>
<td style="text-align:center">BSY</td>
<td style="text-align:center">忙位<br><code>0</code>：空闲<br><code>1</code>：忙</td>
</tr>
</tbody>
</table>
<p>了解了各个端口的作用后，如何从硬盘读取数据似乎就是显而易见的了：</p>
<ol>
<li>设置起始扇区号</li>
<li>设置读取的扇区数量</li>
<li>设置磁盘和访问模式</li>
<li>检测状态位，等待硬盘就绪</li>
<li>从端口读取数据</li>
</ol>
<p>为了方便代码重用 <s>（虽然MBR的代码也不会重用了）</s>，在实验中将读取硬盘的功能包装成一个汇编函数进行调用</p>
<div class="note blue icon-padding flat"><i class="note-icon fas fa-history"></i><p><strong>NASM中的函数调用</strong></p>
<p>调用函数前，通过将参数压栈的方式传递参数<br>
使用<code>call tag</code>调用函数，在执行<code>call</code>指令时会将下一条指令地址压入栈顶<br>
在函数内部，需要使用<code>pushad</code>为主调函数保存寄存器的值<br>
在函数结束后，使用<code>ret</code>返回，在执行<code>ret</code>指令时会取出栈顶值并跳转到该值处执行<br>
调用函数后，通过执行<code>add sp, &lt;imm&gt;</code>操作将栈指针上移，从栈上删除传递的参数</p>
</div>
<p>令函数名为<code>read_sectors</code>，并假定主调函数向其传递四个参数值<code>startSector[15:0]</code>、<code>startSector[27:16]</code>、<code>sectorCount</code>和<code>targetAddress</code>，可以据此先写出函数体</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br></pre></td><td class="code"><pre><span class="line"><span class="symbol">read_sectors:</span></span><br><span class="line">  <span class="comment">; Imagined Stack Appearance</span></span><br><span class="line">  <span class="comment">; ------------------------- (High)</span></span><br><span class="line">  <span class="comment">; targetAddress</span></span><br><span class="line">  <span class="comment">; sectorCount</span></span><br><span class="line">  <span class="comment">; startSector[27:16]</span></span><br><span class="line">  <span class="comment">; startSector[15:0]</span></span><br><span class="line">  <span class="comment">; ret</span></span><br><span class="line">  <span class="comment">; ------------------------- (Lo)</span></span><br><span class="line">  <span class="comment">; Save state</span></span><br><span class="line">  <span class="keyword">push</span> <span class="built_in">sp</span></span><br><span class="line">  <span class="keyword">mov</span> <span class="built_in">bp</span>, <span class="built_in">sp</span></span><br><span class="line">  <span class="keyword">pushad</span></span><br><span class="line"></span><br><span class="line">  <span class="comment">; Fill function here</span></span><br><span class="line"></span><br><span class="line">  <span class="comment">; Restore state</span></span><br><span class="line">  <span class="keyword">popad</span></span><br><span class="line">  <span class="keyword">pop</span> <span class="built_in">sp</span></span><br><span class="line">  <span class="comment">; Return</span></span><br><span class="line">  <span class="keyword">ret</span></span><br></pre></td></tr></table></figure>
<p>之后从栈上依次取出起始扇区号并发送到端口<code>0x1F3~0x1F6</code></p>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：注意正确设置0x1F6端口高4位</strong><br>
0x1F6端口高四位包括了主从硬盘位和访问模式位<br>
如果误使用CHS模式读取硬盘，由于CHS扇区从1开始编号，会导致读出错误的扇区</p>
</div>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：栈上的字长为2字节</strong><br>
在16位模式中栈上的字长为2字节，因此使用<code>bp</code>在栈上取数据时<strong>步进为2而不是4</strong><br>
也就是，如果使用上文中的函数开头，第一个参数的为<code>word [bp + 2 * 2]</code></p>
</div>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">mov</span> <span class="built_in">bx</span>, <span class="built_in">word</span> [<span class="built_in">bp</span> + <span class="number">2</span> * <span class="number">2</span>]   <span class="comment">; bx = startSector[15:0]</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">bl</span>  <span class="comment">; al = startSector[7:0]</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F3</span></span><br><span class="line"></span><br><span class="line"><span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">al</span>, <span class="number">bh</span>  <span class="comment">; al = startSector[15:8]</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F4</span></span><br><span class="line"><span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br><span class="line"></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">bx</span>, <span class="built_in">word</span> [<span class="built_in">bp</span> + <span class="number">3</span> * <span class="number">2</span>]   <span class="comment">; bx = startSector[27:16]</span></span><br><span class="line"><span class="keyword">and</span> <span class="built_in">bx</span>, <span class="number">0x0FFF</span>              <span class="comment">; bx[15:12] = 0</span></span><br><span class="line"><span class="keyword">or</span>  <span class="built_in">bx</span>, <span class="number">0xE000</span>              <span class="comment">; bx[15:12] = 0b1110 (LBA method)</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">bl</span>  <span class="comment">; al = startSector[23:16]</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F5</span></span><br><span class="line"><span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br><span class="line"></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">al</span>, <span class="number">bh</span>  <span class="comment">; al = 0xE:startSector[27:24]</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F6</span></span><br><span class="line"><span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br></pre></td></tr></table></figure>
<p>然后通过<code>0x1F2</code>端口设置读取的扇区数量</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">byte</span> [<span class="built_in">bp</span> + <span class="number">4</span> * <span class="number">2</span>]    <span class="comment">; al = sectorCount</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F2</span></span><br><span class="line"><span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br></pre></td></tr></table></figure>
<p>向<code>0x1F7</code>端口发送<code>0x20</code>指令请求硬盘读</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F7</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">al</span>, <span class="number">0x20</span></span><br><span class="line"><span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br></pre></td></tr></table></figure>
<p>在发送读请求后，通过循环检测<code>0x1F7</code>端口的<code>0</code>、<code>3</code>、<code>7</code>位等待硬盘就绪，就绪的标志为这三位依次为<code>0</code>、<code>1</code>、<code>0</code></p>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：记得等待硬盘就绪</strong><br>
在从端口读取数据前，<strong>一定</strong>要等待硬盘就绪，不然会读出错误的数据</p>
</div>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><span class="line"><span class="symbol">.wait_disk:</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F7</span></span><br><span class="line">    <span class="keyword">in</span>  <span class="built_in">al</span>, <span class="built_in">dx</span></span><br><span class="line">    <span class="keyword">and</span> <span class="built_in">al</span>, <span class="number">0x89</span></span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">al</span>, <span class="number">0x08</span></span><br><span class="line">    <span class="keyword">jnz</span> .wait_disk</span><br></pre></td></tr></table></figure>
<p>最后，从<code>0x1F0</code>端口读取数据</p>
<p><code>0x1F0</code>端口为16位端口，因此一次可以读取两个字节，总的读取次数为</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mtable rowspacing="0.25em" columnalign="right left" columnspacing="0em"><mtr><mtd><mstyle scriptlevel="0" displaystyle="true"><mrow><mi>L</mi><mi>o</mi><mi>o</mi><mi>p</mi><mi>C</mi><mi>o</mi><mi>u</mi><mi>n</mi><mi>t</mi></mrow></mstyle></mtd><mtd><mstyle scriptlevel="0" displaystyle="true"><mrow><mrow></mrow><mo>=</mo><mi>S</mi><mi>e</mi><mi>c</mi><mi>t</mi><mi>o</mi><mi>r</mi><mi>C</mi><mi>o</mi><mi>u</mi><mi>n</mi><mi>t</mi><mo>×</mo><mn>512</mn><mo>÷</mo><mn>2</mn></mrow></mstyle></mtd></mtr><mtr><mtd><mstyle scriptlevel="0" displaystyle="true"><mrow></mrow></mstyle></mtd><mtd><mstyle scriptlevel="0" displaystyle="true"><mrow><mrow></mrow><mo>=</mo><mi>S</mi><mi>e</mi><mi>c</mi><mi>t</mi><mi>o</mi><mi>r</mi><mi>C</mi><mi>o</mi><mi>u</mi><mi>n</mi><mi>t</mi><mo>×</mo><mn>256</mn></mrow></mstyle></mtd></mtr></mtable><annotation encoding="application/x-tex">\begin{aligned}
LoopCount &amp;= SectorCount \times 512 \div 2 \\
          &amp;= SectorCount \times 256
\end{aligned}
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:3em;vertical-align:-1.25em;"></span><span class="mord"><span class="mtable"><span class="col-align-r"><span class="vlist-t vlist-t2"><span class="vlist-r"><span class="vlist" style="height:1.75em;"><span style="top:-3.91em;"><span class="pstrut" style="height:3em;"></span><span class="mord"><span class="mord mathnormal">L</span><span class="mord mathnormal">oo</span><span class="mord mathnormal" style="margin-right:0.07153em;">pC</span><span class="mord mathnormal">o</span><span class="mord mathnormal">u</span><span class="mord mathnormal">n</span><span class="mord mathnormal">t</span></span></span><span style="top:-2.41em;"><span class="pstrut" style="height:3em;"></span><span class="mord"></span></span></span><span class="vlist-s">​</span></span><span class="vlist-r"><span class="vlist" style="height:1.25em;"><span></span></span></span></span></span><span class="col-align-l"><span class="vlist-t vlist-t2"><span class="vlist-r"><span class="vlist" style="height:1.75em;"><span style="top:-3.91em;"><span class="pstrut" style="height:3em;"></span><span class="mord"><span class="mord"></span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">ec</span><span class="mord mathnormal">t</span><span class="mord mathnormal" style="margin-right:0.02778em;">or</span><span class="mord mathnormal" style="margin-right:0.07153em;">C</span><span class="mord mathnormal">o</span><span class="mord mathnormal">u</span><span class="mord mathnormal">n</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mord">512</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">÷</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mord">2</span></span></span><span style="top:-2.41em;"><span class="pstrut" style="height:3em;"></span><span class="mord"><span class="mord"></span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">ec</span><span class="mord mathnormal">t</span><span class="mord mathnormal" style="margin-right:0.02778em;">or</span><span class="mord mathnormal" style="margin-right:0.07153em;">C</span><span class="mord mathnormal">o</span><span class="mord mathnormal">u</span><span class="mord mathnormal">n</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mord">256</span></span></span></span><span class="vlist-s">​</span></span><span class="vlist-r"><span class="vlist" style="height:1.25em;"><span></span></span></span></span></span></span></span></span></span></span></span></p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">xor</span> <span class="built_in">ax</span>, <span class="built_in">ax</span>                  <span class="comment">; Set ax = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">byte</span> [<span class="built_in">bp</span> + <span class="number">4</span> * <span class="number">2</span>]   <span class="comment">; ax = sectorCount</span></span><br><span class="line"><span class="keyword">imul</span> <span class="built_in">ax</span>, <span class="number">256</span>                <span class="comment">; ax = word count</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">bx</span>, <span class="built_in">word</span> [<span class="built_in">bp</span> + <span class="number">5</span> * <span class="number">2</span>]   <span class="comment">; bx = targetAddress</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">si</span>, <span class="number">0</span>                   <span class="comment">; shift = 0</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">cx</span>, <span class="built_in">ax</span>                  <span class="comment">; set loop = ax = word count</span></span><br><span class="line"><span class="symbol">.write_word:</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F0</span></span><br><span class="line">    <span class="keyword">in</span>  <span class="built_in">ax</span>, <span class="built_in">dx</span>              <span class="comment">; read 2 bytes from port</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">word</span> [<span class="built_in">bx</span> + <span class="built_in">si</span>], <span class="built_in">ax</span>  <span class="comment">; write data to memory</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">si</span>, <span class="number">2</span>               <span class="comment">; add 2 bytes for each read operation</span></span><br><span class="line">    <span class="keyword">loop</span> .write_word</span><br></pre></td></tr></table></figure>
<p>所以完整的读取函数为</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br><span class="line">44</span><br><span class="line">45</span><br><span class="line">46</span><br><span class="line">47</span><br><span class="line">48</span><br><span class="line">49</span><br><span class="line">50</span><br><span class="line">51</span><br><span class="line">52</span><br><span class="line">53</span><br><span class="line">54</span><br><span class="line">55</span><br><span class="line">56</span><br><span class="line">57</span><br><span class="line">58</span><br><span class="line">59</span><br><span class="line">60</span><br><span class="line">61</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Loader sector function</span></span><br><span class="line"><span class="comment">; Notice: Loads using LBA method</span></span><br><span class="line"><span class="symbol">read_sectors:</span></span><br><span class="line">    <span class="comment">; push targetAddr</span></span><br><span class="line">    <span class="comment">; push sectorCount</span></span><br><span class="line">    <span class="comment">; push startSector[27:16]</span></span><br><span class="line">    <span class="comment">; push startSector[15:0]</span></span><br><span class="line">    <span class="comment">; push ret</span></span><br><span class="line">    <span class="keyword">push</span> <span class="built_in">sp</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">bp</span>, <span class="built_in">sp</span></span><br><span class="line">    <span class="keyword">pushad</span></span><br><span class="line"></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">bx</span>, <span class="built_in">word</span> [<span class="built_in">bp</span> + <span class="number">2</span> * <span class="number">2</span>]   <span class="comment">; bx = startSector[15:0]</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">bl</span>  <span class="comment">; al = startSector[7:0]</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F3</span></span><br><span class="line">    <span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="number">bh</span>  <span class="comment">; al = startSector[15:8]</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F4</span></span><br><span class="line">    <span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br><span class="line">    </span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">bx</span>, <span class="built_in">word</span> [<span class="built_in">bp</span> + <span class="number">3</span> * <span class="number">2</span>]   <span class="comment">; bx = startSector[27:16]</span></span><br><span class="line">    <span class="keyword">and</span> <span class="built_in">bx</span>, <span class="number">0x0FFF</span>              <span class="comment">; bx[15:12] = 0</span></span><br><span class="line">    <span class="keyword">or</span>  <span class="built_in">bx</span>, <span class="number">0xE000</span>              <span class="comment">; bx[15:12] = 0b1110 (LBA method)</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">bl</span>  <span class="comment">; al = startSector[23:16]</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F5</span></span><br><span class="line">    <span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="number">bh</span>  <span class="comment">; al = 0xE:startSector[27:24]</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F6</span></span><br><span class="line">    <span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br><span class="line"></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">byte</span> [<span class="built_in">bp</span> + <span class="number">4</span> * <span class="number">2</span>]    <span class="comment">; al = sectorCount</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F2</span></span><br><span class="line">    <span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br><span class="line"></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F7</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="number">0x20</span></span><br><span class="line">    <span class="keyword">out</span> <span class="built_in">dx</span>, <span class="built_in">al</span></span><br><span class="line"><span class="symbol"></span></span><br><span class="line"><span class="symbol">    .wait_disk:</span></span><br><span class="line">        <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F7</span></span><br><span class="line">        <span class="keyword">in</span>  <span class="built_in">al</span>, <span class="built_in">dx</span></span><br><span class="line">        <span class="keyword">and</span> <span class="built_in">al</span>, <span class="number">0x89</span></span><br><span class="line">        <span class="keyword">cmp</span> <span class="built_in">al</span>, <span class="number">0x08</span></span><br><span class="line">        <span class="keyword">jnz</span> .wait_disk</span><br><span class="line"></span><br><span class="line">    <span class="keyword">xor</span> <span class="built_in">ax</span>, <span class="built_in">ax</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">byte</span> [<span class="built_in">bp</span> + <span class="number">4</span> * <span class="number">2</span>]   <span class="comment">; ax = sectorCount</span></span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">ax</span>, <span class="number">256</span>    <span class="comment">; ax = word count</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">bx</span>, <span class="built_in">word</span> [<span class="built_in">bp</span> + <span class="number">5</span> * <span class="number">2</span>]   <span class="comment">; bx = targetAddress</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">si</span>, <span class="number">0</span>       <span class="comment">; shift = 0</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">cx</span>, <span class="built_in">ax</span>      <span class="comment">; set loop = ax</span></span><br><span class="line"><span class="symbol">    .write_word:</span></span><br><span class="line">        <span class="keyword">mov</span> <span class="built_in">dx</span>, <span class="number">0x1F0</span></span><br><span class="line">        <span class="keyword">in</span>  <span class="built_in">ax</span>, <span class="built_in">dx</span></span><br><span class="line">        <span class="keyword">mov</span> <span class="built_in">word</span> [<span class="built_in">bx</span> + <span class="built_in">si</span>], <span class="built_in">ax</span></span><br><span class="line">        <span class="keyword">add</span> <span class="built_in">si</span>, <span class="number">2</span></span><br><span class="line">        <span class="keyword">loop</span> .write_word</span><br><span class="line"></span><br><span class="line">    <span class="keyword">popad</span></span><br><span class="line">    <span class="keyword">pop</span> <span class="built_in">sp</span></span><br><span class="line">    <span class="keyword">ret</span></span><br></pre></td></tr></table></figure>
<h3 id="加载BootLoader并跳转">加载BootLoader并跳转</h3>
<p>为了验证上一节中读取硬盘函数的正确性，现在需要在项目中创建BootLoader，由于目前我们还不打算在BootLoader中完成很复杂的任务，所以就先将<a href="https://blog.linloir.cn/2022/07/16/os-journal-vol-2/index.html#Hello-World">上一节中打印’Hello World!'</a>的代码移植到BootLoader中，然后尝试从MBR加载并跳转到BootLoader执行，看看能否正常运行</p>
<p>首先我们需要扩充我们的项目结构：</p>
<ul>
<li>BootLoader自身需要一个汇编文件<code>bootloader.asm</code>，由于它同样属于启动过程，所以可以将它放置在<code>/src/boot/</code>目录下</li>
<li>由于BootLoader加载需要指定 <strong>起始扇区号</strong>、<strong>扇区数量</strong> 和 <strong>加载地址</strong>，为了方便修改，增加代码可读性，可以像C语言那样将这些数值定义为常数值放在头文件中，然后在代码中引用这个头文件。于是我们为启动过程添加一个共同的头文件<code>boot.inc</code>，用于指定启动过程中用到的所有常数值，这个文件同样放置在<code>/src/boot/</code>目录下</li>
</ul>
<p>扩充后的项目结构大致像这样</p>
<figure class="highlight plaintext"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br></pre></td><td class="code"><pre><span class="line">.</span><br><span class="line">├── build</span><br><span class="line">│   └── makefile</span><br><span class="line">├── readme.md</span><br><span class="line">├── run</span><br><span class="line">│   └── hd.img</span><br><span class="line">└── src</span><br><span class="line">    └── boot</span><br><span class="line">        ├── boot.inc</span><br><span class="line">        ├── bootloader.asm</span><br><span class="line">        └── mbr.asm</span><br></pre></td></tr></table></figure>
<p>在开始移植之前，需要先确认BootLoader的起始地址以及它的大小，这涉及到内存地址的安排</p>
<p>在操作系统内核设计的过程中，内存规划是一件令人苦恼的事情，但同时也是一件自由的事情，只要不发生溢出和重叠之类的问题，将内容放置在哪里是一个相对主观的事情</p>
<p>这里我将BootLoader直接放置在MBR后面，占用4个扇区的大小，内存的安排如下</p>
<table>
<thead>
<tr>
<th style="text-align:center">用途</th>
<th style="text-align:center">起始地址</th>
<th style="text-align:center">终止地址</th>
<th style="text-align:center">扇区数</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">MBR</td>
<td style="text-align:center">0x7C00</td>
<td style="text-align:center">0x7E00</td>
<td style="text-align:center">1</td>
</tr>
<tr>
<td style="text-align:center">BootLoader</td>
<td style="text-align:center">0x7E00</td>
<td style="text-align:center">0x8600</td>
<td style="text-align:center">4</td>
</tr>
</tbody>
</table>
<p>完成了内存规划之后，就可以将常数写入头文件中了</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Constants used during the boot procedure</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; MBR</span></span><br><span class="line">BOOTLOADER_SECTOR_START_15_0 <span class="built_in">equ</span> <span class="number">1</span></span><br><span class="line">BOOTLOADER_SECTOR_START_27_16 <span class="built_in">equ</span> <span class="number">0</span></span><br><span class="line">BOOTLOADER_SECTOR_COUNT <span class="built_in">equ</span> <span class="number">4</span></span><br><span class="line">BOOTLOADER_LOAD_ADDRESS <span class="built_in">equ</span> <span class="number">0x7E00</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; BootLoader</span></span><br><span class="line">GDT_START_ADDRESS <span class="built_in">equ</span> <span class="number">0x9000</span></span><br><span class="line">ZERO_SELECTOR <span class="built_in">equ</span> <span class="number">0x00</span></span><br><span class="line">KERNEL_CODE_SELECTOR <span class="built_in">equ</span> <span class="number">0x08</span></span><br><span class="line">KERNEL_DATA_SELECTOR <span class="built_in">equ</span> <span class="number">0x10</span></span><br><span class="line">KERNEL_STACK_SELECTOR <span class="built_in">equ</span> <span class="number">0x18</span></span><br><span class="line">GDT_SIZE <span class="built_in">equ</span> <span class="number">4</span></span><br></pre></td></tr></table></figure>
<p>打印<code>Hello World!</code>代码的移植步骤就比较简单了，直接复制粘贴入<code>bootloader.asm</code>即可，不过需要注意的是要修改一下头部伪代码中的<code>[org 0x7C00]</code>为BootLoader的起始地址<code>[org 0x7E00]</code></p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br><span class="line">44</span><br><span class="line">45</span><br></pre></td><td class="code"><pre><span class="line"><span class="meta">%include</span> <span class="string">&quot;boot.inc&quot;</span></span><br><span class="line">[org <span class="number">0x7E00</span>]</span><br><span class="line">[<span class="meta">bits</span> <span class="number">16</span>]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Print something</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="number">0xB800</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">gs</span>, <span class="built_in">ax</span>  <span class="comment">; Set video segment</span></span><br><span class="line"></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">si</span>, <span class="number">0</span></span><br><span class="line"><span class="symbol">print:</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dl</span>, <span class="built_in">byte</span> [_msg + <span class="built_in">si</span>]</span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">dl</span>, <span class="number">0</span></span><br><span class="line">    <span class="keyword">je</span>  print_exit</span><br><span class="line">    <span class="keyword">inc</span> <span class="built_in">si</span></span><br><span class="line">    <span class="comment">; Calculate cordinate in vga memory</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [_row]</span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">ax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [_col]</span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">ax</span>, <span class="number">2</span></span><br><span class="line">    <span class="comment">; Copy character to memory</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">bx</span>, <span class="built_in">ax</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">gs</span>:<span class="built_in">bx</span>], <span class="built_in">dl</span>    <span class="comment">; Character to be printed</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">gs</span>:<span class="built_in">bx</span> + <span class="number">1</span>], <span class="number">0x0F</span>  <span class="comment">; Black background with white foreground</span></span><br><span class="line">    <span class="comment">; Set new cordinate</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">word</span> [_col], <span class="number">1</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="built_in">word</span> [_col]</span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">ax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">jne</span> add_row_exit</span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">word</span> [_row], <span class="number">1</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">word</span> [_col], <span class="number">0</span></span><br><span class="line"><span class="symbol">    add_row_exit:</span></span><br><span class="line">    <span class="comment">; Reture to top</span></span><br><span class="line">    <span class="keyword">jmp</span> print</span><br><span class="line"><span class="symbol">print_exit:</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Halt here</span></span><br><span class="line"><span class="keyword">jmp</span> $</span><br><span class="line"></span><br><span class="line"><span class="comment">; Variables</span></span><br><span class="line">_row    <span class="built_in">dw</span> <span class="number">0</span></span><br><span class="line">_col    <span class="built_in">dw</span> <span class="number">0</span></span><br><span class="line">_msg    <span class="built_in">db</span> <span class="string">&#x27;Hello World!&#x27;</span>,</span><br><span class="line">        <span class="built_in">db</span> <span class="number">0</span></span><br><span class="line"></span><br><span class="line"><span class="built_in">times</span> <span class="number">2048</span> - ($ - $$) <span class="built_in">db</span> <span class="number">1</span></span><br></pre></td></tr></table></figure>
<p>之后，对<code>mbr.asm</code>中的代码进行修改</p>
<p>向其中添加<code>read_sectors</code>函数并且调用<code>read_sectors</code>函数</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Read bootloader</span></span><br><span class="line"><span class="keyword">push</span> BOOTLOADER_LOAD_ADDRESS</span><br><span class="line"><span class="keyword">push</span> BOOTLOADER_SECTOR_COUNT</span><br><span class="line"><span class="keyword">push</span> BOOTLOADER_SECTOR_START_27_16</span><br><span class="line"><span class="keyword">push</span> BOOTLOADER_SECTOR_START_15_0</span><br><span class="line"><span class="keyword">call</span> read_sectors</span><br><span class="line"><span class="keyword">add</span> <span class="built_in">sp</span>, <span class="number">8</span></span><br></pre></td></tr></table></figure>
<p>跳转到BootLoader的起始地址</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Jump to bootloader</span></span><br><span class="line"><span class="keyword">jmp</span> BOOTLOADER_LOAD_ADDRESS</span><br></pre></td></tr></table></figure>
<p>最后，修改<code>makefile</code>文件，由于添加了两个文件，依赖关系也产生了变化，新的依赖关系如下</p>
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
<td style="text-align:center"><code>mbr.bin</code><br><code>bootloader.bin</code></td>
</tr>
<tr>
<td style="text-align:center"><code>mbr.bin</code></td>
<td style="text-align:center">MBR编译后的二进制文件</td>
<td style="text-align:center"><code>mbr.asm</code><br><code>boot.inc</code></td>
</tr>
<tr>
<td style="text-align:center"><code>bootloader.bin</code></td>
<td style="text-align:center">BootLoader编译后的二进制文件</td>
<td style="text-align:center"><code>bootloader.asm</code><br><code>boot.inc</code></td>
</tr>
<tr>
<td style="text-align:center"><code>mbr.asm</code></td>
<td style="text-align:center">MBR源文件</td>
<td style="text-align:center">-</td>
</tr>
<tr>
<td style="text-align:center"><code>bootloader.asm</code></td>
<td style="text-align:center">BootLoader源文件</td>
<td style="text-align:center">-</td>
</tr>
<tr>
<td style="text-align:center"><code>boot.inc</code></td>
<td style="text-align:center">头文件</td>
<td style="text-align:center">-</td>
</tr>
</tbody>
</table>
<p>根据新的依赖关系，可以编写新的<code>makefile</code>文件如下，其中除了添加了新的依赖文件编译规则外，还添加了对头文件路径的指定</p>
<figure class="highlight makefile"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br></pre></td><td class="code"><pre><span class="line">ASM_COMPILER := nasm</span><br><span class="line"></span><br><span class="line">SRC_PATH := ../src</span><br><span class="line">RUN_PATH := ../run</span><br><span class="line">INCLUDE_PATH := <span class="variable">$(<span class="built_in">addprefix</span> -I, $(<span class="built_in">dir</span> $(<span class="built_in">shell</span> find &#x27;<span class="variable">$(SRC_PATH)</span>&#x27; -name &#x27;*.inc&#x27;)</span>))</span><br><span class="line"></span><br><span class="line"><span class="meta"><span class="keyword">.PHONY</span>:</span></span><br><span class="line"><span class="section">build: mbr.bin bootloader.bin</span></span><br><span class="line">  qemu-img create <span class="variable">$(RUN_PATH)</span>/hd.img 10m</span><br><span class="line">  dd if=mbr.bin of=<span class="variable">$(RUN_PATH)</span>/hd.img bs=512 count=1 seek=0 conv=notrunc</span><br><span class="line">  dd if=bootloader.bin of=<span class="variable">$(RUN_PATH)</span>/hd.img bs=512 count=4 seek=1 conv=notrunc</span><br><span class="line"></span><br><span class="line"><span class="section">%.bin: <span class="variable">$(SRC_PATH)</span>/boot/%.asm</span></span><br><span class="line">  <span class="variable">$(ASM_COMPILER)</span> -o <span class="variable">$@</span> -f bin <span class="variable">$(INCLUDE_PATH)</span> <span class="variable">$^</span></span><br><span class="line"></span><br><span class="line"><span class="meta"><span class="keyword">.PHONY</span>:</span></span><br><span class="line"><span class="section">clean:</span></span><br><span class="line">  rm -f *.o* *.bin</span><br><span class="line">  rm -f <span class="variable">$(RUN_PATH)</span>/*.img</span><br><span class="line"></span><br><span class="line"><span class="meta"><span class="keyword">.PHONY</span>:</span></span><br><span class="line"><span class="section">run:</span></span><br><span class="line">  qemu-system-i386 -hda <span class="variable">$(RUN_PATH)</span>/hd.img -vga virtio -serial null -parallel stdio -no-reboot</span><br></pre></td></tr></table></figure>
<p>运行的方式与上一章相同，进入<code>build/</code>目录下执行<code>make clean build run</code>即可以对代码进行编译和运行</p>
<p>运行的结果应该与上一章一样，在屏幕的第一行打印出<code>Hello World!</code>字样</p>
<p><img src="/img/os-journal-vol-3/IMG_20220721-194409347.png" alt="图 2"></p>
<h2 id="在BootLoader中开启保护模式">在BootLoader中开启保护模式</h2>
<h3 id="知识准备">知识准备</h3>
<div class="note blue icon-padding flat"><i class="note-icon fas fa-history"></i><p><strong>保护模式</strong></p>
<p>保护模式是目前英特尔处理器主流的运行模式，在保护模式中，处理器以32位模式运行，所有的寄存器也都为32位，因此程序可以访问到<span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><msup><mn>2</mn><mn>32</mn></msup><mo>=</mo><mn>4</mn><mi>G</mi><mi>i</mi><mi>B</mi></mrow><annotation encoding="application/x-tex">2^{32} = 4GiB</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.8141em;"></span><span class="mord"><span class="mord">2</span><span class="msupsub"><span class="vlist-t"><span class="vlist-r"><span class="vlist" style="height:0.8141em;"><span style="top:-3.063em;margin-right:0.05em;"><span class="pstrut" style="height:2.7em;"></span><span class="sizing reset-size6 size3 mtight"><span class="mord mtight"><span class="mord mtight">32</span></span></span></span></span></span></span></span></span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.6833em;"></span><span class="mord">4</span><span class="mord mathnormal">G</span><span class="mord mathnormal">i</span><span class="mord mathnormal" style="margin-right:0.05017em;">B</span></span></span></span>的内存空间</p>
<p>保护模式提出了 <strong>段</strong> 的概念，在CPU产生地址以后，会判断这个地址是否超出了段所规定的地址，从而避免程序之间的越界访问</p>
<p>除此之外，保护模式还包括了特权级保护等，在后面的实验中会逐个涉及</p>
</div>
<p>上面提到，在保护模式中提出了段的概念，在保护模式中运行的各种代码都需要声明自己所使用的段，并且由CPU监视地址的访问是否越界。那么，代码是如何告诉CPU自己所使用的段信息，CPU又是如何知道段的界限的呢？</p>
<p>实际上，为了能够让CPU得知段的具体信息，<strong>需要在内存中分配一段地址空间，在其中放置关于各个段的说明信息，这部分空间就被叫做全局描述符表GDT <em>(Global Descriptor Table)</em></strong></p>
<p>全局描述符表的<strong>起始地址与它的大小组合成一段48位长度的数据提供给CPU作为查询全局描述符表基地址的依据，放置在CPU中的GDTR寄存器中</strong>，GDTR的结构如下</p>
<table>
<thead>
<tr>
<th style="text-align:center">位区间</th>
<th style="text-align:center">描述</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">[48:16]</td>
<td style="text-align:center">基地址 <em>(Offset)</em> [31:0]</td>
</tr>
<tr>
<td style="text-align:center">[15:0]</td>
<td style="text-align:center">大小 <em>(Size)</em> [15:0]</td>
</tr>
</tbody>
</table>
<p>其中，大小由最多能够放入的描述符个数<code>Count</code>指定</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mi>S</mi><mi>i</mi><mi>z</mi><mi>e</mi><mo>=</mo><mi>C</mi><mi>o</mi><mi>u</mi><mi>n</mi><mi>t</mi><mo>×</mo><mn>8</mn><mo>−</mo><mn>1</mn></mrow><annotation encoding="application/x-tex">Size = Count \times 8 - 1
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.6833em;"></span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">i</span><span class="mord mathnormal">ze</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal" style="margin-right:0.07153em;">C</span><span class="mord mathnormal">o</span><span class="mord mathnormal">u</span><span class="mord mathnormal">n</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.7278em;vertical-align:-0.0833em;"></span><span class="mord">8</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">−</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.6444em;"></span><span class="mord">1</span></span></span></span></span></p>
<p>全局描述符表看起来就像一个数组，其存放的都是64位长度的元素，<strong>每一个元素都用来描述一个段的信息，被称作段描述符 <em>(Segment Descriptor)</em></strong>，全局描述符表中的第一个元素始终为空描述符，从第二个元素开始可以由程序定义</p>
<p>对于段描述符而言，它由七个部分组成，结构如下</p>
<table>
<thead>
<tr>
<th style="text-align:center">位区间</th>
<th style="text-align:center">描述</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">[63:56]</td>
<td style="text-align:center">起始地址 <em>(Base)</em> [31:24]</td>
</tr>
<tr>
<td style="text-align:center">[55:52]</td>
<td style="text-align:center">标志位 <em>(Flags)</em> [3:0]</td>
</tr>
<tr>
<td style="text-align:center">[51:48]</td>
<td style="text-align:center">段界限 <em>(Limit)</em> [19:16]</td>
</tr>
<tr>
<td style="text-align:center">[47:40]</td>
<td style="text-align:center">访问控制位 <em>(Access Byte)</em> [7:0]</td>
</tr>
<tr>
<td style="text-align:center">[39:32]</td>
<td style="text-align:center">起始地址 <em>(Base)</em> [23:16]</td>
</tr>
<tr>
<td style="text-align:center">[31:16]</td>
<td style="text-align:center">起始地址 <em>(Base)</em> [15:0]</td>
</tr>
<tr>
<td style="text-align:center">[15:0]</td>
<td style="text-align:center">段界限 <em>(Limit)</em> [15:0]</td>
</tr>
</tbody>
</table>
<p>其中，访问控制位又分为七个不同的部分，其结构和作用如下</p>
<table>
<thead>
<tr>
<th style="text-align:center">位</th>
<th style="text-align:center">缩写</th>
<th style="text-align:center">描述</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">0</td>
<td style="text-align:center">A</td>
<td style="text-align:center"><strong>访问位 <em>(Accessed bit)</em></strong><br><code>0</code>：段未被访问<br><code>1</code>：段在上次清除这个访问位后被访问过</td>
</tr>
<tr>
<td style="text-align:center">1</td>
<td style="text-align:center">RW</td>
<td style="text-align:center"><strong>读写权限位 <em>(Readable/Writable bit)</em></strong><br><strong>代码段：</strong><code>0</code>不可读，<code>1</code>可读，始终不可写<br><strong>数据段：</strong><code>0</code>不可写，<code>1</code>可写，始终可读</td>
</tr>
<tr>
<td style="text-align:center">2</td>
<td style="text-align:center">DC</td>
<td style="text-align:center"><strong>增长方向/一致性标志位 <em>(Direction/Conforming bit)</em></strong><br><strong>代码段：</strong><code>0</code>只能由DPL中指定的特权级执行，<code>1</code>代表DPL中指定可执行的最高特权级<br><strong>数据段：</strong><code>0</code>段向高地址增长，<code>1</code>段向低地址增长</td>
</tr>
<tr>
<td style="text-align:center">3</td>
<td style="text-align:center">E</td>
<td style="text-align:center"><strong>可执行位 <em>(Executable bit)</em></strong><br><code>0</code>：不可执行，为数据段<br><code>1</code>：可执行，为代码段</td>
</tr>
<tr>
<td style="text-align:center">4</td>
<td style="text-align:center">S</td>
<td style="text-align:center"><strong>描述符类型位 <em>(Descriptor type bit)</em></strong><br><code>0</code>：系统段，例如<code>TSS</code><br><code>1</code>：代码段或是数据段</td>
</tr>
<tr>
<td style="text-align:center">5-6</td>
<td style="text-align:center">DPL</td>
<td style="text-align:center"><strong>描述符特权级 <em>(Descriptor privilege level field)</em></strong><br><code>0</code>：最高特权级（内核）<br><code>3</code>：最低特权级（用户）</td>
</tr>
<tr>
<td style="text-align:center">7</td>
<td style="text-align:center">P</td>
<td style="text-align:center"><strong>存在位 <em>(Present bit)</em></strong><br><code>0</code>：该描述符不可用 <em>(invalid)</em><br><code>1</code>：描述符可用</td>
</tr>
</tbody>
</table>
<p>段描述符的标志位则使用三个不同的标志设置了段的粒度、位模式</p>
<table>
<thead>
<tr>
<th style="text-align:center">位</th>
<th style="text-align:center">缩写</th>
<th style="text-align:center">描述</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">0</td>
<td style="text-align:center">Reserved</td>
<td style="text-align:center">保留位，始终置<code>0</code></td>
</tr>
<tr>
<td style="text-align:center">1</td>
<td style="text-align:center">L</td>
<td style="text-align:center"><strong>长模式标志位 <em>(Long-mode code flag)</em></strong><br><code>0</code>：位模式由DB位指定<br><code>1</code>：段描述的位64位代码段</td>
</tr>
<tr>
<td style="text-align:center">2</td>
<td style="text-align:center">DB</td>
<td style="text-align:center"><strong>位模式位 <em>(Size flag)</em></strong><br><code>0</code>：描述符对应16位保护模式段<br><code>1</code>：描述符对应32位保护模式段<br>置位时<code>L</code>位应为<code>0</code></td>
</tr>
<tr>
<td style="text-align:center">3</td>
<td style="text-align:center">G</td>
<td style="text-align:center"><strong>粒度标志位 <em>(Granularity flag)</em></strong><br><code>0</code>：描述符中的段界限按照字节单位计算<br><code>1</code>：描述符中的段界限按照4KiB单位计算</td>
</tr>
</tbody>
</table>
<p>在了解了段描述符各个位的作用后，CPU又是如何通过这些地址和标志位去判断地址的合法性的呢？要明确这个问题，就需要首先知道CPU是如何生成地址的</p>
<p>进入保护模式之后，每当CPU生成一个地址，它实际上生成的是相对于段的偏移地址<code>Offset</code>，CPU同时还会通过上文中提到的段选择子从全局描述符表中取出段的基地址<code>Base</code>，之后CPU中的地址变换部件会判断偏移地址的合法性，并组合偏移地址和基地址得到真实的物理地址<code>Address</code>，这个最后的物理地址才会用于访问内存数据</p>
<p>也就是说，<code>Address</code>、<code>Base</code>和<code>Offset</code>之间存在如下关系</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mi>A</mi><mi>d</mi><mi>d</mi><mi>r</mi><mi>e</mi><mi>s</mi><mi>s</mi><mo>=</mo><mi>B</mi><mi>a</mi><mi>s</mi><mi>e</mi><mo>+</mo><mi>O</mi><mi>f</mi><mi>f</mi><mi>s</mi><mi>e</mi><mi>t</mi></mrow><annotation encoding="application/x-tex">Address = Base + Offset
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.6944em;"></span><span class="mord mathnormal">A</span><span class="mord mathnormal">dd</span><span class="mord mathnormal">ress</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal" style="margin-right:0.05017em;">B</span><span class="mord mathnormal">a</span><span class="mord mathnormal">se</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal" style="margin-right:0.02778em;">O</span><span class="mord mathnormal" style="margin-right:0.10764em;">ff</span><span class="mord mathnormal">se</span><span class="mord mathnormal">t</span></span></span></span></span></p>
<p>而CPU对地址合法性的检验，实际上就是在做<code>Offset</code>和段描述符中界限<code>Limit</code>之间关系的判断，当然，由于粒度<code>Granularity</code>的引入，在计算实际的界限时还需要掺入粒度单位</p>
<p>对于向上增长的段，它的偏移地址需要满足</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mn>0</mn><mo>≤</mo><mi>O</mi><mi>f</mi><mi>f</mi><mi>s</mi><mi>e</mi><mi>t</mi><mo>≤</mo><mi>L</mi><mi>i</mi><mi>m</mi><mi>i</mi><mi>t</mi><mo>×</mo><mi>G</mi><mi>r</mi><mi>a</mi><mi>n</mi><mi>u</mi><mi>l</mi><mi>a</mi><mi>r</mi><mi>i</mi><mi>t</mi><mi>y</mi></mrow><annotation encoding="application/x-tex">0 \le Offset \le Limit \times Granularity
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.7804em;vertical-align:-0.136em;"></span><span class="mord">0</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">≤</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal" style="margin-right:0.02778em;">O</span><span class="mord mathnormal" style="margin-right:0.10764em;">ff</span><span class="mord mathnormal">se</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">≤</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal">L</span><span class="mord mathnormal">imi</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal">G</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">an</span><span class="mord mathnormal">u</span><span class="mord mathnormal" style="margin-right:0.01968em;">l</span><span class="mord mathnormal">a</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">i</span><span class="mord mathnormal">t</span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span></span></span></span></span></p>
<p>这个很好理解，因为 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mn>0</mn></mrow><annotation encoding="application/x-tex">0</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.6444em;"></span><span class="mord">0</span></span></span></span> 代表着段中第一个字节的地址，而 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mi>L</mi><mi>i</mi><mi>m</mi><mi>i</mi><mi>t</mi><mo>×</mo><mi>G</mi><mi>r</mi><mi>a</mi><mi>n</mi><mi>u</mi><mi>l</mi><mi>a</mi><mi>r</mi><mi>i</mi><mi>t</mi><mi>y</mi></mrow><annotation encoding="application/x-tex">Limit \times Granularity</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal">L</span><span class="mord mathnormal">imi</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal">G</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">an</span><span class="mord mathnormal">u</span><span class="mord mathnormal" style="margin-right:0.01968em;">l</span><span class="mord mathnormal">a</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">i</span><span class="mord mathnormal">t</span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span></span></span></span> 代表段中最后一个字节的地址，偏移量需要介于这两者之间，才是合法的访问</p>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：粒度不是直接进行乘4KiB的操作</strong><br>
实际上，<span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mi>L</mi><mi>i</mi><mi>m</mi><mi>i</mi><mi>t</mi><mo>×</mo><mi>G</mi><mi>r</mi><mi>a</mi><mi>n</mi><mi>u</mi><mi>l</mi><mi>a</mi><mi>r</mi><mi>i</mi><mi>t</mi><mi>y</mi></mrow><annotation encoding="application/x-tex">Limit \times Granularity</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal">L</span><span class="mord mathnormal">imi</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal">G</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">an</span><span class="mord mathnormal">u</span><span class="mord mathnormal" style="margin-right:0.01968em;">l</span><span class="mord mathnormal">a</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">i</span><span class="mord mathnormal">t</span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span></span></span></span> 并不像我们想的那样将界限直接左移12位，而是相当于<code>Limit &lt;&lt; 12 | 0xFFF</code><br>
也就是说，在4KiB粒度下，<code>0xFFFFF</code>的界限实际对应着<code>0xFFFFFFFF</code>，<code>0x00000</code>对应着<code>0x00000FFF</code></p>
</div>
<p>对于向下增长的段，则稍微有些不一样，一个比较容易理解的阐述是，<strong>对于使用相同基址和界限的向上增长段，其合法地址在向下增长的段中不合法，其不合法的地址在向下增长的段中合法</strong>，如果使用公式来更严谨的解释这一说法，则偏移地址需要满足</p>
<p><span class="katex-display"><span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><semantics><mrow><mi>L</mi><mi>i</mi><mi>m</mi><mi>i</mi><mi>t</mi><mo>×</mo><mi>G</mi><mi>r</mi><mi>a</mi><mi>n</mi><mi>u</mi><mi>l</mi><mi>a</mi><mi>r</mi><mi>i</mi><mi>t</mi><mi>y</mi><mo>+</mo><mn>1</mn><mo>≤</mo><mi>O</mi><mi>f</mi><mi>f</mi><mi>s</mi><mi>e</mi><mi>t</mi><mo>≤</mo><mn>0</mn><mi mathvariant="monospace">x</mi><mi>F</mi><mi>F</mi><mi>F</mi><mi>F</mi><mi>F</mi><mi>F</mi><mi>F</mi><mi>F</mi><mtext>  </mtext><mo stretchy="false">(</mo><mn>0</mn><mi mathvariant="monospace">x</mi><mi>F</mi><mi>F</mi><mi>F</mi><mi>F</mi><mtext>  </mtext><mi>f</mi><mi>o</mi><mi>r</mi><mtext>  </mtext><mn>16</mn><mtext>  </mtext><mi>b</mi><mi>i</mi><mi>t</mi><mo stretchy="false">)</mo></mrow><annotation encoding="application/x-tex">Limit \times Granularity + 1 \le Offset \le 0\mathtt{x}FFFFFFFF\;(0\mathtt{x}FFFF\;for\;16\;bit)
</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal">L</span><span class="mord mathnormal">imi</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal">G</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">an</span><span class="mord mathnormal">u</span><span class="mord mathnormal" style="margin-right:0.01968em;">l</span><span class="mord mathnormal">a</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">i</span><span class="mord mathnormal">t</span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.7804em;vertical-align:-0.136em;"></span><span class="mord">1</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">≤</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal" style="margin-right:0.02778em;">O</span><span class="mord mathnormal" style="margin-right:0.10764em;">ff</span><span class="mord mathnormal">se</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">≤</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:1em;vertical-align:-0.25em;"></span><span class="mord">0</span><span class="mord mathtt">x</span><span class="mord mathnormal" style="margin-right:0.13889em;">FFFFFFFF</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mopen">(</span><span class="mord">0</span><span class="mord mathtt">x</span><span class="mord mathnormal" style="margin-right:0.13889em;">FFFF</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord mathnormal" style="margin-right:0.10764em;">f</span><span class="mord mathnormal" style="margin-right:0.02778em;">or</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord">16</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mord mathnormal">bi</span><span class="mord mathnormal">t</span><span class="mclose">)</span></span></span></span></span></p>
<p>举一个例子，假设需要在16位模式下设置一个1KiB栈段，其最高地址为<code>0xFFFF</code>，则其最低地址为<code>0xFC00</code>，大家可能很快就能想到，将基地址设置为<code>0x0000</code>并设置界限为<code>0xFC00</code>既可以描述这个栈段</p>
<p>但且慢，注意到偏移地址的最低合法地址为 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mi>L</mi><mi>i</mi><mi>m</mi><mi>i</mi><mi>t</mi><mo>×</mo><mi>G</mi><mi>r</mi><mi>a</mi><mi>n</mi><mi>u</mi><mi>l</mi><mi>a</mi><mi>r</mi><mi>i</mi><mi>t</mi><mi>y</mi><mo>+</mo><mn>1</mn></mrow><annotation encoding="application/x-tex">Limit \times Granularity + 1</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal">L</span><span class="mord mathnormal">imi</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.8889em;vertical-align:-0.1944em;"></span><span class="mord mathnormal">G</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">an</span><span class="mord mathnormal">u</span><span class="mord mathnormal" style="margin-right:0.01968em;">l</span><span class="mord mathnormal">a</span><span class="mord mathnormal" style="margin-right:0.02778em;">r</span><span class="mord mathnormal">i</span><span class="mord mathnormal">t</span><span class="mord mathnormal" style="margin-right:0.03588em;">y</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.6444em;"></span><span class="mord">1</span></span></span></span>，如果界限设置为<code>0xFC00</code>，则最低合法地址为 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mn>0</mn><mi mathvariant="monospace">x</mi><mn>0000</mn><mo>+</mo><mn>0</mn><mi mathvariant="monospace">x</mi><mi>F</mi><mi>C</mi><mn>00</mn><mo>+</mo><mn>1</mn><mo>=</mo><mn>0</mn><mi mathvariant="monospace">x</mi><mi>F</mi><mi>C</mi><mn>01</mn></mrow><annotation encoding="application/x-tex">0\mathtt{x}0000 + 0\mathtt{x}FC00 + 1 = 0\mathtt{x}FC01</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.7278em;vertical-align:-0.0833em;"></span><span class="mord">0</span><span class="mord mathtt">x</span><span class="mord">0000</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord">0</span><span class="mord mathtt">x</span><span class="mord mathnormal" style="margin-right:0.07153em;">FC</span><span class="mord">00</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">+</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.6444em;"></span><span class="mord">1</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.6833em;"></span><span class="mord">0</span><span class="mord mathtt">x</span><span class="mord mathnormal" style="margin-right:0.07153em;">FC</span><span class="mord">01</span></span></span></span>，栈的大小则不是1KiB，而是1KiB - 1了</p>
<p>因此，界限应该设置为<code>0xFBFF</code>才能让栈段的可用地址为1KiB</p>
<div class="note info flat"><p><strong>全局描述符表不仅可以存储段描述符</strong><br>
在之后的章节中，会了解到全局描述符表中不仅可以存放段描述符，还可以存放其他长度相同的描述符，例如任务状态段TSS <em>(Task State Segment)</em>，这将会在实现用户进程的章节介绍</p>
</div>
<p>既然是数组，那就意味着可以通过下标（索引）来取出某个位置上的元素，CPU也正是这么做的，当一个程序向CPU声明自己所使用的段时，它实际上是向CPU提供了一个索引，<strong>这个16位索引中存储了其所使用的段在全局描述符表中的下标以及特权级信息（将会在用户进程的实现中介绍），被称作段选择子 <em>(Segment Selector)</em></strong></p>
<p>段选择子由其对应的段在全局描述符表中的索引、描述符表类型以及特权级构成，其结构如下</p>
<table>
<thead>
<tr>
<th style="text-align:center">位区间</th>
<th style="text-align:center">描述</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">[15:3]</td>
<td style="text-align:center"><strong>索引 <em>(Index)</em></strong> [12:0]</td>
</tr>
<tr>
<td style="text-align:center">[2:2]</td>
<td style="text-align:center"><strong>描述符表类型 <em>(TI)</em></strong><br><code>0</code>：使用全局描述符表 <em>(GDT)</em><br><code>1</code>：使用局部描述符表 <em>(LDT)</em></td>
</tr>
<tr>
<td style="text-align:center">[1:0]</td>
<td style="text-align:center"><strong>特权级 <em>(RPL)</em></strong> [1:0]</td>
</tr>
</tbody>
</table>
<p><strong>我们目前只需要设置索引位</strong></p>
<p>由于在实验中不会使用到局部描述符表，因此在此处不会介绍它，这一位直接设置为<code>0</code></p>
<p>同时<strong>特权级位会在后面涉及到用户进程时再次介绍，目前我们只需要关心<code>RPL = 0</code>的段</strong></p>
<hr>
<p>总的来说，操作系统内的代码<strong>首先在内存中的全局描述符表内声明需要用到的段</strong>，之后<strong>将对应段的选择子提供给CPU</strong>，CPU在产生地址后，<strong>根据选择子中的信息在全局描述符表中查询段的具体信息</strong>来判断访问是否越界</p>
<h3 id="进入保护模式">进入保护模式</h3>
<p>在了解了保护模式以及全局描述符表的相关知识后，就可以编写代码在BootLoader中开启保护模式了</p>
<div class="note blue icon-padding flat"><i class="note-icon fas fa-history"></i><p><strong>进入保护模式</strong><br>
进入保护模式有五个步骤，分别是 <strong>设置全局描述符表</strong>、<strong>关闭中断</strong>， <strong>开启第21根地址线</strong>、<strong>打开保护模式开关</strong> 和 <strong>执行一次远跳转送入代码段选择子</strong></p>
</div>
<p><strong>首先进行第一步：设置全局描述符表</strong></p>
<p>通过上一小节可以知道，全局描述符表实际上就是存放在内存中的一段类似于数组的空间，但是目前我们还没有指定这一段空间的位置，所以要先进行内存规划</p>
<p>实际上位置的指定相对自由，在实验中将GDT放置在<code>0x9000-0x10000</code>的位置处</p>
<table>
<thead>
<tr>
<th style="text-align:center">用途</th>
<th style="text-align:center">起始地址</th>
<th style="text-align:center">终止地址</th>
<th style="text-align:center">扇区数</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">MBR</td>
<td style="text-align:center">0x7C00</td>
<td style="text-align:center">0x7E00</td>
<td style="text-align:center">1</td>
</tr>
<tr>
<td style="text-align:center">BootLoader</td>
<td style="text-align:center">0x7E00</td>
<td style="text-align:center">0x8600</td>
<td style="text-align:center">4</td>
</tr>
<tr>
<td style="text-align:center">GDT</td>
<td style="text-align:center">0x9000</td>
<td style="text-align:center">0x10000</td>
<td style="text-align:center">-</td>
</tr>
</tbody>
</table>
<p>将新的常数写入头文件中方便调用</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; BootLoader</span></span><br><span class="line">GDT_START_ADDRESS <span class="built_in">equ</span> <span class="number">0x9000</span></span><br></pre></td></tr></table></figure>
<p>之后，在BootLoader中向GDT中添加元素，在实验中我们使用平坦模式，所以只需要添加代码段、数据段以及栈段的描述符，以及最初的空描述符</p>
<div class="note info flat"><p><strong>平坦模式</strong><br>
平坦模式就是所有的段都对所有地址空间有完整的访问权限，并且所有的程序使用同一个代码段，简化了地址的访问<br>
之所以使用这个模式是因为在后面的实验中，会有另一套地址管理的机制，称为分页机制，地址的保护将会使用分页机制进行</p>
</div>
<p>可以根据段特性来设置描述符的值</p>
<table>
<thead>
<tr>
<th style="text-align:center">段</th>
<th style="text-align:center">起始地址</th>
<th style="text-align:center">段界限</th>
<th style="text-align:center">A</th>
<th style="text-align:center">RW</th>
<th style="text-align:center">DC</th>
<th style="text-align:center">E</th>
<th style="text-align:center">S</th>
<th style="text-align:center">DPL</th>
<th style="text-align:center">P</th>
<th style="text-align:center">L</th>
<th style="text-align:center">DB</th>
<th style="text-align:center">G</th>
<th style="text-align:center">描述符值</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">空</td>
<td style="text-align:center">0x0</td>
<td style="text-align:center">0x0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0x00000000_00000000</td>
</tr>
<tr>
<td style="text-align:center">代码</td>
<td style="text-align:center">0x0</td>
<td style="text-align:center">0xFFFFFFFF</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0x00CF9A00_0000FFFF</td>
</tr>
<tr>
<td style="text-align:center">数据</td>
<td style="text-align:center">0x0</td>
<td style="text-align:center">0xFFFFFFFF</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0x00CF9200_0000FFFF</td>
</tr>
<tr>
<td style="text-align:center">栈</td>
<td style="text-align:center">0x0</td>
<td style="text-align:center">0x0</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0</td>
<td style="text-align:center">1</td>
<td style="text-align:center">1</td>
<td style="text-align:center">0x00CF9600_0000FFFF</td>
</tr>
</tbody>
</table>
<p>确定了描述符值以后，使用<code>mov</code>命令放置在给GDT分配的内存空间中</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Initialize Global Descriptor Table</span></span><br><span class="line"><span class="comment">; 0x0 Empty selector</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">0</span> * <span class="number">4</span>], <span class="number">0x00000000</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">1</span> * <span class="number">4</span>], <span class="number">0x00000000</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; 0x08 Kernel code selector</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">2</span> * <span class="number">4</span>], <span class="number">0x0000FFFF</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">3</span> * <span class="number">4</span>], <span class="number">0x00CF9A00</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; 0x10 Kernel data selector</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">4</span> * <span class="number">4</span>], <span class="number">0x0000FFFF</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">5</span> * <span class="number">4</span>], <span class="number">0x00CF9200</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; 0x18 Kernel stack selector</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">6</span> * <span class="number">4</span>], <span class="number">0x0000FFFF</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">7</span> * <span class="number">4</span>], <span class="number">0x00CF9600</span></span><br></pre></td></tr></table></figure>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：注意小端模式</strong><br>
在IA-32处理器中，数据的存储使用的是小端模式<br>
这就意味着对于一个64位的数据，<strong>低地址存放低32位，高地址存放高32位</strong><br>
虽然对于低32位，其存放顺序同样是小端模式，但是具体怎么放是<code>mov</code>指令的事情，我们就不需要关心了，我们只需要知道在使用<code>mov</code>指令的时候，<strong>宏观上要把低32位放在低地址上</strong>就对了</p>
</div>
<p>根据写入的顺序就可以设置描述符对应的选择子了，由于我们只使用GDT而且只考虑特权级为<code>0</code>的情况，因此选择子的低3位均为<code>0</code></p>
<p>将选择子以及描述符个数写入头文件中方便重用</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br></pre></td><td class="code"><pre><span class="line">ZERO_SELECTOR <span class="built_in">equ</span> <span class="number">0x00</span></span><br><span class="line">KERNEL_CODE_SELECTOR <span class="built_in">equ</span> <span class="number">0x08</span></span><br><span class="line">KERNEL_DATA_SELECTOR <span class="built_in">equ</span> <span class="number">0x10</span></span><br><span class="line">KERNEL_STACK_SELECTOR <span class="built_in">equ</span> <span class="number">0x18</span></span><br><span class="line">GDT_SIZE <span class="built_in">equ</span> <span class="number">4</span></span><br></pre></td></tr></table></figure>
<p>设置好了GDT里的内容后，就要考虑如何将其装载入<code>GDTR</code>了</p>
<p>由于<code>GDTR</code>为糟糕的48位长度，很显然没有一个寄存器能够放下这样长度的数据，因此工程师们又想出了一个天秀的方式：<strong>先把要放入GDTR的值放置在内存的一段地址中，然后使用<code>lgdt</code>命令读取这个地址，由该指令将地址之后的48位数据拷贝到GDTR中</strong></p>
<p>所以首先指定一段用于存放GDTR数据的内存</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Variables</span></span><br><span class="line">gdt_descriptor  <span class="built_in">dw</span> <span class="number">0</span>,</span><br><span class="line">                <span class="built_in">dd</span> GDT_START_ADDRESS</span><br></pre></td></tr></table></figure>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：注意小端模式</strong><br>
在汇编的变量声明中，先声明的数据位于低地址<br>
由于IA-32处理器又是小端模式，所以<strong>GDTR中的Size段位于低地址，应该先声明，然后再声明其Base段</strong></p>
</div>
<p>在BootLoader中由 <span class="katex"><span class="katex-mathml"><math xmlns="http://www.w3.org/1998/Math/MathML"><semantics><mrow><mi>S</mi><mi>i</mi><mi>z</mi><mi>e</mi><mo>=</mo><mi>C</mi><mi>o</mi><mi>u</mi><mi>n</mi><mi>t</mi><mo>×</mo><mn>8</mn><mo>−</mo><mn>1</mn></mrow><annotation encoding="application/x-tex">Size = Count \times 8 - 1</annotation></semantics></math></span><span class="katex-html" aria-hidden="true"><span class="base"><span class="strut" style="height:0.6833em;"></span><span class="mord mathnormal" style="margin-right:0.05764em;">S</span><span class="mord mathnormal">i</span><span class="mord mathnormal">ze</span><span class="mspace" style="margin-right:0.2778em;"></span><span class="mrel">=</span><span class="mspace" style="margin-right:0.2778em;"></span></span><span class="base"><span class="strut" style="height:0.7667em;vertical-align:-0.0833em;"></span><span class="mord mathnormal" style="margin-right:0.07153em;">C</span><span class="mord mathnormal">o</span><span class="mord mathnormal">u</span><span class="mord mathnormal">n</span><span class="mord mathnormal">t</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">×</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.7278em;vertical-align:-0.0833em;"></span><span class="mord">8</span><span class="mspace" style="margin-right:0.2222em;"></span><span class="mbin">−</span><span class="mspace" style="margin-right:0.2222em;"></span></span><span class="base"><span class="strut" style="height:0.6444em;"></span><span class="mord">1</span></span></span></span> 这一公式来设置GDT的大小</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Set table size</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">word</span> [gdt_descriptor], GDT_SIZE * <span class="number">8</span> - <span class="number">1</span></span><br></pre></td></tr></table></figure>
<p>一切准备妥当后，就是用<code>lgdt</code>指令将这糟糕的48位长度数据送入寄存器</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Load gdt</span></span><br><span class="line"><span class="keyword">lgdt</span> [gdt_descriptor]</span><br></pre></td></tr></table></figure>
<p><strong>然后进行关中断操作</strong></p>
<p>之所以要关中断，是因为在保护模式下中断的实现与实模式下不一样，在我们尚未实现中断的时候，贸然进入保护模式会产生错误。因此我们先在BootLoader中关闭中断，当我们在后续章节中建立起完善的中断机制之后再打开中断</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">cli</span>   <span class="comment">; Disable interrupt</span></span><br></pre></td></tr></table></figure>
<p>在关闭中断后，就可以进行第三步：<strong>开启第21根地址线</strong></p>
<p>在实模式下，CPU始终将21根地址线置为低电平，这样不论指令寄存器如何自增，始终会因为溢出而在<code>0xFFFFF</code>处回到<code>0x00000</code><br>
为了进入保护模式，就需要解除这一层封印，让寻址突破20位限制</p>
<p>在编写BootLoader中的代码前，可以先看一段有趣的代码来感受CPU是如何通过第21根地址线将地址限制在20位的</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br><span class="line">44</span><br><span class="line">45</span><br><span class="line">46</span><br><span class="line">47</span><br><span class="line">48</span><br><span class="line">49</span><br><span class="line">50</span><br><span class="line">51</span><br><span class="line">52</span><br><span class="line">53</span><br><span class="line">54</span><br><span class="line">55</span><br><span class="line">56</span><br><span class="line">57</span><br><span class="line">58</span><br><span class="line">59</span><br><span class="line">60</span><br><span class="line">61</span><br><span class="line">62</span><br><span class="line">63</span><br><span class="line">64</span><br><span class="line">65</span><br><span class="line">66</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; The following code is public domain licensed</span></span><br><span class="line"> </span><br><span class="line">[<span class="meta">bits</span> <span class="number">16</span>]</span><br><span class="line"> </span><br><span class="line"><span class="comment">; Function: check_a20</span></span><br><span class="line"><span class="comment">;</span></span><br><span class="line"><span class="comment">; Purpose: to check the status of the a20 line in a completely self-contained state-preserving way.</span></span><br><span class="line"><span class="comment">;          The function can be modified as necessary by removing push&#x27;s at the beginning and their</span></span><br><span class="line"><span class="comment">;          respective pop&#x27;s at the end if complete self-containment is not required.</span></span><br><span class="line"><span class="comment">;</span></span><br><span class="line"><span class="comment">; Returns: 0 in ax if the a20 line is disabled (memory wraps around)</span></span><br><span class="line"><span class="comment">;          1 in ax if the a20 line is enabled (memory does not wrap around)</span></span><br><span class="line"><span class="symbol"> </span></span><br><span class="line"><span class="symbol">check_a20:</span></span><br><span class="line">    <span class="keyword">pushf</span></span><br><span class="line">    <span class="keyword">push</span> <span class="built_in">ds</span></span><br><span class="line">    <span class="keyword">push</span> <span class="built_in">es</span></span><br><span class="line">    <span class="keyword">push</span> <span class="built_in">di</span></span><br><span class="line">    <span class="keyword">push</span> <span class="built_in">si</span></span><br><span class="line"> </span><br><span class="line">    <span class="keyword">cli</span>                   <span class="comment">; Disable interrupt</span></span><br><span class="line"> </span><br><span class="line">    <span class="comment">; Save state----------</span></span><br><span class="line">    <span class="keyword">xor</span> <span class="built_in">ax</span>, <span class="built_in">ax</span> <span class="comment">; ax = 0</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">es</span>, <span class="built_in">ax</span></span><br><span class="line"> </span><br><span class="line">    <span class="keyword">not</span> <span class="built_in">ax</span> <span class="comment">; ax = 0xFFFF</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">ds</span>, <span class="built_in">ax</span></span><br><span class="line"> </span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">di</span>, <span class="number">0x0500</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">si</span>, <span class="number">0x0510</span></span><br><span class="line"> </span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">byte</span> [<span class="built_in">es</span>:<span class="built_in">di</span>]</span><br><span class="line">    <span class="keyword">push</span> <span class="built_in">ax</span></span><br><span class="line"> </span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">al</span>, <span class="built_in">byte</span> [<span class="built_in">ds</span>:<span class="built_in">si</span>]</span><br><span class="line">    <span class="keyword">push</span> <span class="built_in">ax</span></span><br><span class="line">    <span class="comment">; End save state-------</span></span><br><span class="line"></span><br><span class="line">    <span class="comment">; Main code------------</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">es</span>:<span class="built_in">di</span>], <span class="number">0x00</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">ds</span>:<span class="built_in">si</span>], <span class="number">0xFF</span>  <span class="comment">; If trunc, the addr should be (0xFFFF0 + 0x510) &amp; 0xFFFFF = 0x500 otherwise should be 0x100500</span></span><br><span class="line"> </span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">byte</span> [<span class="built_in">es</span>:<span class="built_in">di</span>], <span class="number">0xFF</span></span><br><span class="line">    <span class="comment">; End main code--------</span></span><br><span class="line"></span><br><span class="line">    <span class="comment">; Restore state--------</span></span><br><span class="line">    <span class="keyword">pop</span> <span class="built_in">ax</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">ds</span>:<span class="built_in">si</span>], <span class="built_in">al</span></span><br><span class="line"> </span><br><span class="line">    <span class="keyword">pop</span> <span class="built_in">ax</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">es</span>:<span class="built_in">di</span>], <span class="built_in">al</span></span><br><span class="line"> </span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="number">0</span></span><br><span class="line">    <span class="keyword">je</span> check_a20__exit</span><br><span class="line"> </span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">ax</span>, <span class="number">1</span></span><br><span class="line"><span class="symbol"> </span></span><br><span class="line"><span class="symbol">check_a20__exit:</span></span><br><span class="line">    <span class="keyword">pop</span> <span class="built_in">si</span></span><br><span class="line">    <span class="keyword">pop</span> <span class="built_in">di</span></span><br><span class="line">    <span class="keyword">pop</span> <span class="built_in">es</span></span><br><span class="line">    <span class="keyword">pop</span> <span class="built_in">ds</span></span><br><span class="line">    <span class="keyword">popf</span></span><br><span class="line"> </span><br><span class="line">    <span class="keyword">ret</span></span><br></pre></td></tr></table></figure>
<p>至于打开这根地址线，由于该地址线由南桥A20端口控制，端口号为<code>0x92</code>，控制位位于第二位，因此代码编写如下</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Open A20</span></span><br><span class="line"><span class="keyword">in</span>  <span class="built_in">al</span>, <span class="number">0x92</span></span><br><span class="line"><span class="keyword">or</span>  <span class="built_in">al</span>, <span class="number">0b0000_0010</span> <span class="comment">; Set A20 enabled</span></span><br><span class="line"><span class="keyword">out</span> <span class="number">0x92</span>, <span class="built_in">al</span></span><br></pre></td></tr></table></figure>
<p>接下来就来到了<strong>保护模式真正的开关，CR0寄存器中的PE位</strong></p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Set PE</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">cr0</span></span><br><span class="line"><span class="keyword">or</span>  <span class="built_in">eax</span>, <span class="number">1</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">cr0</span>, <span class="built_in">eax</span></span><br></pre></td></tr></table></figure>
<p>最后<strong>执行一次远跳转，送入代码段选择子，正式开启保护模式</strong>，其中<code>protected_mode_begin</code>部分将在<a href="#%E7%AC%AC%E4%BA%8C%E4%B8%AAhello-world">下一小节</a>中完成</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">jmp</span> <span class="built_in">dword</span> KERNEL_CODE_SELECTOR:protected_mode_begin</span><br></pre></td></tr></table></figure>
<div class="note info flat"><p><strong>远跳转</strong><br>
远跳转由两个部分组成：段选择子和段内偏移，<strong>执行远跳转会将段选择子送入代码段寄存器CS，同时将段内偏移送入EIP</strong>，使得CPU在新的代码段的基地址上以新的段内偏移开始执行指令<br>
由于代码段选择子 <strong>不能够手动设置</strong>，因此只能够通过远跳转进行设置，所以<strong>此处执行的远跳转主要目的是为了送入代码段选择子</strong></p>
</div>
<h3 id="第二个Hello-World">第二个Hello World</h3>
<p>进入保护模式以后，自然要输出些什么才能确认前面的代码都正确无误地执行了。因此，在这一小节中就来实现保护模式中的第二个<code>Hello World!</code></p>
<p>由于进入保护模式以后，代码全面进入了32位模式，所以也要添加对应的伪代码</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Protected mode starts here</span></span><br><span class="line"></span><br><span class="line">[<span class="meta">bits</span> <span class="number">32</span>] <span class="comment">; Indicates codes below run in 32-bit mode</span></span><br><span class="line"><span class="symbol"></span></span><br><span class="line"><span class="symbol">protected_mode_begin:</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Fill something later</span></span><br></pre></td></tr></table></figure>
<p>这时，我们还没有设置好各个段选择子，因此应当尽快将数据段和栈段的选择子送入寄存器，由于暂时用不到附加的段寄存器，所以不妨将它们设置为空描述符</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Set selectors</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, KERNEL_DATA_SELECTOR</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ds</span>, <span class="built_in">ax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, KERNEL_STACK_SELECTOR</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ss</span>, <span class="built_in">ax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, ZERO_SELECTOR</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">es</span>, <span class="built_in">ax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">fs</span>, <span class="built_in">ax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">gs</span>, <span class="built_in">ax</span></span><br></pre></td></tr></table></figure>
<p>接着就可以移植原先实模式下的代码，鉴于实模式与保护模式地区别，在移植时需要进行如下修改</p>
<ul>
<li>寄存器由16位更换到32位</li>
<li>已然可以访问32位地址，不再需要<code>[gs:bx]</code>这样的访问模式</li>
</ul>
<div class="note red icon-padding flat"><i class="note-icon fas fa-exclamation-circle"></i><p><strong>陷阱：不是在32位模式下所有的寄存器都要使用32位</strong><br>
寄存器位宽的使用始终要符合数据的宽度，例如从内存取出字节的时候就应该使用寄存器的8位模式作为操作数，而不能一味地使用32位寄存器</p>
</div>
<p>移植后的代码如下</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment">; Print something</span></span><br><span class="line"><span class="keyword">xor</span> <span class="built_in">eax</span>, <span class="built_in">eax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ebx</span>, <span class="number">0xB8000</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">esi</span>, <span class="number">0</span></span><br><span class="line"><span class="symbol">print:</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dl</span>, <span class="built_in">byte</span> [_msg + <span class="built_in">esi</span>]</span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">dl</span>, <span class="number">0</span></span><br><span class="line">    <span class="keyword">je</span>  print_exit</span><br><span class="line">    <span class="keyword">inc</span> <span class="built_in">esi</span></span><br><span class="line">    <span class="comment">; Calculate cordinate in vga memory</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [_row]</span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">eax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [_col]</span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">eax</span>, <span class="number">2</span></span><br><span class="line">    <span class="comment">; Copy character to memory</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">edi</span>, <span class="built_in">eax</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">ebx</span> + <span class="built_in">edi</span>], <span class="built_in">dl</span>    <span class="comment">; Character to be printed</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">ebx</span> + <span class="built_in">edi</span> + <span class="number">1</span>], <span class="number">0x0F</span>  <span class="comment">; Black background with white foreground</span></span><br><span class="line">    <span class="comment">; Set new cordinate</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">dword</span> [_col], <span class="number">1</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [_col]</span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">eax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">jne</span> add_row_exit</span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">dword</span> [_row], <span class="number">1</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dword</span> [_col], <span class="number">0</span></span><br><span class="line"><span class="symbol">    add_row_exit:</span></span><br><span class="line">    <span class="comment">; Reture to top</span></span><br><span class="line">    <span class="keyword">jmp</span> print</span><br><span class="line"><span class="symbol">print_exit:</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Halt here</span></span><br><span class="line"><span class="keyword">jmp</span> $</span><br><span class="line"></span><br><span class="line"><span class="comment">; Variables</span></span><br><span class="line">_row    <span class="built_in">dd</span> <span class="number">0</span></span><br><span class="line">_col    <span class="built_in">dd</span> <span class="number">0</span></span><br><span class="line">_msg    <span class="built_in">db</span> <span class="string">&#x27;Hello World!&#x27;</span>,</span><br><span class="line">        <span class="built_in">db</span> <span class="number">0</span></span><br></pre></td></tr></table></figure>
<p>完整的<code>bootloader.asm</code>如下</p>
<figure class="highlight x86asm"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br><span class="line">18</span><br><span class="line">19</span><br><span class="line">20</span><br><span class="line">21</span><br><span class="line">22</span><br><span class="line">23</span><br><span class="line">24</span><br><span class="line">25</span><br><span class="line">26</span><br><span class="line">27</span><br><span class="line">28</span><br><span class="line">29</span><br><span class="line">30</span><br><span class="line">31</span><br><span class="line">32</span><br><span class="line">33</span><br><span class="line">34</span><br><span class="line">35</span><br><span class="line">36</span><br><span class="line">37</span><br><span class="line">38</span><br><span class="line">39</span><br><span class="line">40</span><br><span class="line">41</span><br><span class="line">42</span><br><span class="line">43</span><br><span class="line">44</span><br><span class="line">45</span><br><span class="line">46</span><br><span class="line">47</span><br><span class="line">48</span><br><span class="line">49</span><br><span class="line">50</span><br><span class="line">51</span><br><span class="line">52</span><br><span class="line">53</span><br><span class="line">54</span><br><span class="line">55</span><br><span class="line">56</span><br><span class="line">57</span><br><span class="line">58</span><br><span class="line">59</span><br><span class="line">60</span><br><span class="line">61</span><br><span class="line">62</span><br><span class="line">63</span><br><span class="line">64</span><br><span class="line">65</span><br><span class="line">66</span><br><span class="line">67</span><br><span class="line">68</span><br><span class="line">69</span><br><span class="line">70</span><br><span class="line">71</span><br><span class="line">72</span><br><span class="line">73</span><br><span class="line">74</span><br><span class="line">75</span><br><span class="line">76</span><br><span class="line">77</span><br><span class="line">78</span><br><span class="line">79</span><br><span class="line">80</span><br><span class="line">81</span><br><span class="line">82</span><br><span class="line">83</span><br><span class="line">84</span><br><span class="line">85</span><br><span class="line">86</span><br><span class="line">87</span><br><span class="line">88</span><br><span class="line">89</span><br><span class="line">90</span><br><span class="line">91</span><br><span class="line">92</span><br><span class="line">93</span><br><span class="line">94</span><br><span class="line">95</span><br><span class="line">96</span><br><span class="line">97</span><br><span class="line">98</span><br><span class="line">99</span><br><span class="line">100</span><br><span class="line">101</span><br><span class="line">102</span><br></pre></td><td class="code"><pre><span class="line"><span class="meta">%include</span> <span class="string">&quot;boot.inc&quot;</span></span><br><span class="line">[org <span class="number">0x7E00</span>]</span><br><span class="line">[<span class="meta">bits</span> <span class="number">16</span>]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Disable interrupts</span></span><br><span class="line"><span class="keyword">cli</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Initialize Global Descriptor Table</span></span><br><span class="line"><span class="comment">; 0x0 Empty selector</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">0</span> * <span class="number">4</span>], <span class="number">0x00000000</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">1</span> * <span class="number">4</span>], <span class="number">0x00000000</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; 0x08 Kernel code selector</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">2</span> * <span class="number">4</span>], <span class="number">0x0000FFFF</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">3</span> * <span class="number">4</span>], <span class="number">0x00CF9A00</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; 0x10 Kernel data selector</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">4</span> * <span class="number">4</span>], <span class="number">0x0000FFFF</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">5</span> * <span class="number">4</span>], <span class="number">0x00CF9200</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; 0x18 Kernel stack selector</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">6</span> * <span class="number">4</span>], <span class="number">0x0000FFFF</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">dword</span> [GDT_START_ADDRESS + <span class="number">7</span> * <span class="number">4</span>], <span class="number">0x00CF9600</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Set table size</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">word</span> [gdt_descriptor], GDT_SIZE * <span class="number">8</span> - <span class="number">1</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Load gdt</span></span><br><span class="line"><span class="keyword">lgdt</span> [gdt_descriptor]</span><br><span class="line"></span><br><span class="line"><span class="comment">; Open A20</span></span><br><span class="line"><span class="keyword">in</span>  <span class="built_in">al</span>, <span class="number">0x92</span></span><br><span class="line"><span class="keyword">or</span>  <span class="built_in">al</span>, <span class="number">0b0000_0010</span></span><br><span class="line"><span class="keyword">out</span> <span class="number">0x92</span>, <span class="built_in">al</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Set PE</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">cr0</span></span><br><span class="line"><span class="keyword">or</span>  <span class="built_in">eax</span>, <span class="number">1</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">cr0</span>, <span class="built_in">eax</span></span><br><span class="line"></span><br><span class="line"><span class="keyword">jmp</span> <span class="built_in">dword</span> KERNEL_CODE_SELECTOR:protected_mode_begin</span><br><span class="line"></span><br><span class="line"><span class="comment">; Halt here</span></span><br><span class="line"><span class="keyword">jmp</span> $</span><br><span class="line"></span><br><span class="line"><span class="comment">; Variables</span></span><br><span class="line">gdt_descriptor  <span class="built_in">dw</span> <span class="number">0</span>,</span><br><span class="line">                <span class="built_in">dd</span> GDT_START_ADDRESS</span><br><span class="line"></span><br><span class="line">[<span class="meta">bits</span> <span class="number">32</span>]</span><br><span class="line"><span class="symbol"></span></span><br><span class="line"><span class="symbol">protected_mode_begin:</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Set selectors</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, KERNEL_DATA_SELECTOR</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ds</span>, <span class="built_in">ax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, KERNEL_STACK_SELECTOR</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ss</span>, <span class="built_in">ax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ax</span>, ZERO_SELECTOR</span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">es</span>, <span class="built_in">ax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">fs</span>, <span class="built_in">ax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">gs</span>, <span class="built_in">ax</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Print something</span></span><br><span class="line"><span class="keyword">xor</span> <span class="built_in">eax</span>, <span class="built_in">eax</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">ebx</span>, <span class="number">0xB8000</span></span><br><span class="line"><span class="keyword">mov</span> <span class="built_in">esi</span>, <span class="number">0</span></span><br><span class="line"><span class="symbol">print:</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dl</span>, <span class="built_in">byte</span> [_msg + <span class="built_in">esi</span>]</span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">dl</span>, <span class="number">0</span></span><br><span class="line">    <span class="keyword">je</span>  print_exit</span><br><span class="line">    <span class="keyword">inc</span> <span class="built_in">esi</span></span><br><span class="line">    <span class="comment">; Calculate cordinate in vga memory</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [_row]</span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">eax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [_col]</span><br><span class="line">    <span class="keyword">imul</span> <span class="built_in">eax</span>, <span class="number">2</span></span><br><span class="line">    <span class="comment">; Copy character to memory</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">edi</span>, <span class="built_in">eax</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">ebx</span> + <span class="built_in">edi</span>], <span class="built_in">dl</span>    <span class="comment">; Character to be printed</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">byte</span> [<span class="built_in">ebx</span> + <span class="built_in">edi</span> + <span class="number">1</span>], <span class="number">0x0F</span>  <span class="comment">; Black background with white foreground</span></span><br><span class="line">    <span class="comment">; Set new cordinate</span></span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">dword</span> [_col], <span class="number">1</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">eax</span>, <span class="built_in">dword</span> [_col]</span><br><span class="line">    <span class="keyword">cmp</span> <span class="built_in">eax</span>, <span class="number">80</span></span><br><span class="line">    <span class="keyword">jne</span> add_row_exit</span><br><span class="line">    <span class="keyword">add</span> <span class="built_in">dword</span> [_row], <span class="number">1</span></span><br><span class="line">    <span class="keyword">mov</span> <span class="built_in">dword</span> [_col], <span class="number">0</span></span><br><span class="line"><span class="symbol">    add_row_exit:</span></span><br><span class="line">    <span class="comment">; Reture to top</span></span><br><span class="line">    <span class="keyword">jmp</span> print</span><br><span class="line"><span class="symbol">print_exit:</span></span><br><span class="line"></span><br><span class="line"><span class="comment">; Halt here</span></span><br><span class="line"><span class="keyword">jmp</span> $</span><br><span class="line"></span><br><span class="line"><span class="comment">; Variables</span></span><br><span class="line">_row    <span class="built_in">dd</span> <span class="number">0</span></span><br><span class="line">_col    <span class="built_in">dd</span> <span class="number">0</span></span><br><span class="line">_msg    <span class="built_in">db</span> <span class="string">&#x27;Hello World!&#x27;</span>,</span><br><span class="line">        <span class="built_in">db</span> <span class="number">0</span></span><br><span class="line"></span><br></pre></td></tr></table></figure>
<div class="note success flat"><p><strong>完成</strong><br>
至此，就完成了本章的全部任务，赶紧使用<code>make clean build run</code>来测试代码的运行情况吧！</p>
</div>

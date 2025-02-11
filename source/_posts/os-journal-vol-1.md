---
title: 操统实验日志 第一章 序章
date: 2022-07-15 08:36:46
description: "简述: 在一切开始之前，请允许我先简要地介绍一下关于这个实验的一切。这一系列日志将是我对大二下学期操作系统实验课程中实验的一个整体回顾与记录。当然，在我写下这些的时候，我还完全不知道这份日志可以做到怎样的完成度，但我仍希望在我写下这些文字的暑假里能够以一系列详实、可复现的日志作为我对过去这一个学期里的这样一门有趣的课程的一个交代。我希望，通过这一系列日志，可以记录我完成一个简单操作系统的全部过程、记录在完成操作系统实验中可能遇到的难题和它们的解决方案，并作为一份简易操作系统的简易教程，帮助我自己或是别人复现这个实验。那么，准备好了吗？我们开始吧！"
tags:
  - 操作系统
  - 大学
categories:
  - 技术
---

<h2 id="简述">简述</h2>
<div class="note info flat"><p>在一切开始之前，请允许我先简要地介绍一下关于这个实验的一切</p>
</div>
<h3 id="它是关于什么的">它是关于什么的</h3>
<p>这一系列日志将是我对大二下学期<strong>操作系统实验</strong>课程中实验的一个整体回顾与记录。</p>
<p>当然，在我写下这些的时候，我还完全不知道这份日志可以做到怎样的完成度，但我仍希望在我写下这些文字的暑假里能够以一系列详实、可复现的日志作为我对过去这一个学期里的这样一门有趣的课程的一个交代。</p>
<h3 id="它有什么用">它有什么用</h3>
<p>我希望，通过这一系列日志，可以</p>
<ul>
<li>记录我完成一个简单操作系统的全部过程</li>
<li>记录在完成操作系统实验中可能遇到的难题和它们的解决方案</li>
<li>作为一份简易操作系统的简易教程，帮助我自己或是别人复现这个实验</li>
</ul>
<div class="note success flat"><p>那么，准备好了吗？<br>
我们开始吧！</p>
</div>
<h2 id="环境搭建">环境搭建</h2>
<p>在介绍具体的安装方法前，先在此列出所有需要的环境，方便快速对照安装</p>
<h3 id="必须的">必须的</h3>
<div class="note red icon-padding flat"><i class="note-icon fas fa-warning"></i><p>由于实验的需要，以下环境是<strong>必须准备</strong>的</p>
</div>
<ul>
<li><strong>Linux环境</strong>：由于代码编译和虚拟机运行需要依赖<code>Linux</code>环境，因此需要准备一个<code>Linux</code>设备，在我的实现中使用了<code>VMWare WorkStation</code>与<code>Ubuntu 20.04 LTS</code>的组合</li>
<li><strong>QEMU</strong>：安装在<code>Linux</code>环境下的虚拟机，用来运行编写的操作系统</li>
<li><strong>C/C++编译环境</strong>：代码编译需要，下列不完全
<ul>
<li><strong>CMake</strong>：编写<code>makefile</code>需要</li>
<li><strong>GCC/LLVM</strong>：编译器，在我最初的实现中使用了<code>GCC</code>，但是由于操作系统涉及到很多*bare-metal C++*也就是脱离<code>C/C++</code>标准库的代码，<code>GCC</code>在编译的时候会遇到一些棘手的问题，这些问题当然在<code>LLVM</code>中也会遇到，但相对<code>GCC</code>更好解决一些，故在我的实现中最后改为了使用<code>LLVM</code>作为编译器</li>
</ul>
</li>
<li><strong>NASM</strong>：汇编代码编译器</li>
</ul>
<h3 id="建议的">建议的</h3>
<div class="note warning flat"><p>为了提升在实验过程中的编写、调试等体验，以下环境是<strong>可选</strong>的，并且<strong>在我的实现中安装</strong>的</p>
</div>
<p>在<code>Windows</code>下</p>
<ul>
<li><strong>VSCode</strong>：现代IDE，可选安装如下插件
<ul>
<li><strong>ASM Code Lens或其他asm插件</strong>：提供<code>asm</code>语法高亮</li>
<li><strong>C/C++</strong>：提供<code>C/C++</code>语法高亮</li>
<li><strong>CMake, CMake Language Support, CMake Tools</strong>：提供<code>CMake</code>支持</li>
<li><strong>Hex Editor</strong>：调试时查看生成的二进制文件</li>
<li><strong>GitLens</strong>：优秀的<code>Git</code>可视化管理工具，提供了比<code>VSCode</code>原生更丰富的功能，包括分支等</li>
<li><strong>Remote-SSH等Remote家族</strong>：提供远程连接<code>Linux</code>环境支持，使得可以在<code>Windows</code>环境下直接编写<code>Linux</code>环境中的代码并且进行编译等基本操作</li>
</ul>
</li>
<li><strong>Windows 11</strong>：提供了更漂亮的<code>PWS/CMD</code>工具</li>
<li><strong>Windows终端</strong>：<code>Microsoft Store</code>中提供的开源的漂亮的命令行工具，用于连接<code>Linux shell</code></li>
<li><strong>IDA</strong>：在调试bug的时候用于打开<code>.o</code>文件看看里面发生了什么奇怪的错误</li>
</ul>
<p>在<code>Linux</code>下</p>
<ul>
<li><strong>git</strong>：不多解释了，基本上啥都要用到这个</li>
<li><strong>ZSH</strong>：终端美化
<ul>
<li><strong>powerlevel10k</strong>：个人使用的主题</li>
</ul>
</li>
<li><strong>pwndbg</strong>：好用的<code>GDB</code>工具</li>
</ul>
<h3 id="Ubuntu安装指南">Ubuntu安装指南</h3>
<div class="note info flat"><p>Ubuntu版本以进行实验时的20.04LTS版本为例，其他版本的安装方式基本相同</p>
</div>
<div class="tabs" id="ubuntu_installation"><ul class="nav-tabs"><button type="button" data-href="#ubuntu_installation-1" class="tab active">下载</button><button type="button" data-href="#ubuntu_installation-2" class="tab">虚拟机配置</button><button type="button" data-href="#ubuntu_installation-3" class="tab">换源</button><button type="button" data-href="#ubuntu_installation-4" class="tab">软件环境配置</button><button type="button" data-href="#ubuntu_installation-5" class="tab">设置SSH</button></ul><div class="tab-contents"><div class="tab-item-content active" id="ubuntu_installation-1"><p>前往<a target="_blank" rel="noopener" href="https://ubuntu.com/download/desktop">Ubuntu官网</a>处下载对应版本的镜像文件</p>
<p><img src="/img/os-journal-vol-1/IMG_20220716-113134498.png" alt="镜像下载页"></p>
<div class="note info flat"><p>下载页面会随时间改变，如果需要下载旧版本Ubuntu，可以进入<a target="_blank" rel="noopener" href="https://ubuntu.com/download/alternative-downloads">alternative downloads</a>页面下载</p>
</div>
<p>下载得到一个文件名如<code>ubuntu-20.04.4-desktop-amd64.iso</code>的镜像文件</p>
<div class="note success flat"><p>有了镜像文件以后，就可以进入下一步，在虚拟机中使用该镜像安装系统了</p>
</div>
<!-- ![图 2](/assets/os-journal-vol-1/IMG_20220716-115254230.png)   --><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="ubuntu_installation-2"><div class="note info flat"><p>由于我在实验中使用的是VMWare Workstation，此处以<strong>VMWare Workstation Pro 16</strong>为例进行介绍，其他虚拟机可以使用<a target="_blank" rel="noopener" href="https://www.google.com">搜索引擎</a>查询对应Linux配置方式</p>
</div>
<p>在VMWare Workstation中安装Ubuntu的步骤如下：</p>
<ol>
<li>创建新的虚拟机</li>
<li>配置选择<code>自定义（高级）</code></li>
<li>硬件兼容性选择<code>Workstation 16.2.x</code></li>
<li>选择<code>稍后安装操作系统</code></li>
<li>安装的系统选择<code>Linux</code>及<code>Ubuntu 64位</code></li>
<li>设置名称位置</li>
<li>根据CPU设置处理器内核数</li>
<li>内存大小设置为<code>4~8GiB</code></li>
<li>网络类型设置为<code>NAT</code></li>
<li>I/O控制器类型选择为<code>LSI Logic</code></li>
<li>虚拟磁盘类型选择<code>SCSI</code></li>
<li>磁盘选择<code>创建新虚拟磁盘</code></li>
<li>自由设置其他内容</li>
</ol>
<p>硬件配置如下</p>
<p><img src="/img/os-journal-vol-1/IMG_20220716-160607671.png" alt="图 4"></p>
<p>为了从上一步下载的镜像文件中安装系统，需要编辑虚拟机设置，在CD/DVD的连接栏选择使用ISO映像文件并设置为上一步中下载的镜像文件，同时勾选启动时连接</p>
<div class="note info flat"><p>如果设备中没有找到CD/DVD，需要通过下方添加添加一个CD/DVD驱动器</p>
</div>
<p>在启动虚拟机之前，由于在某些情况下虚拟机的显示分辨率会太低导致显示不出安装界面的下一步按钮，因此在启动之前可以在虚拟机设置中调整显示器分辨率。</p>
<p>这里将显示器设备中的<strong>监视器</strong>一栏设置为<strong>指定监视器设置</strong>并设置最大分辨率为当前显示器的分辨率</p>
<div class="note info flat"><p>如果这一步没有产生效果，并且在安装时确实遇到了无法点击按钮的情况，可以在第一步选择<code>Try Ubuntu</code>进入桌面</p>
<p>之后右键在<code>Display Settings</code>中设置分辨率和缩放</p>
<p>如果没有合适的缩放，可以开启<code>Fractional Scaling</code>进行小数位缩放，可以多出125%、150%和175%的缩放比设置</p>
</div>
<p>设置好后启动虚拟机，进入<code>.iso</code>文件中的系统，启动后会自动进入Ubuntu安装程序</p>
<div class="note info flat"><p>如果这一步无法启动，检查CD/DVD是否设置为启动时连接</p>
</div>
<p>安装步骤如下：</p>
<ol>
<li>安装程序中选择<code>Install Ubuntu</code>进入安装</li>
<li>检查键盘布局是否正确，具体测试方式为在输入栏中测试键盘各个键输入是否符合预期</li>
<li>选择<code>Minimal installation</code>进行最小安装，同时可以取消勾选<code>Download updates while installing Ubuntu</code>，避免因为遇到无法访问的情况卡住安装，更新可以在后续换源后进行，这里可以不着急</li>
<li>简便起见，在<code>Installation type</code>步骤可以直接选择<code>Erase disk and install Ubuntu</code>，如果有自己的分区喜好，可以选择<code>Something else</code>，具体分区方式<a target="_blank" rel="noopener" href="https://www.google.com">网上</a>有完备介绍，可以跟随教程设置，完成后选择<code>Install Now</code></li>
<li>时区选择<code>Shanghai</code></li>
<li>设置用户名密码</li>
<li>点选<code>Continue</code>执行安装即可</li>
</ol>
<p>在安装完成之后，系统会提示是否重启系统，由于需要关闭CD/DVD驱动器以免再次进入安装系统，在这一步不进行重启系统，关闭该通知进入系统桌面，右上角菜单选择关机，关机过程中会有一个提示移出安装媒介并按回车键，直接按回车关机即可</p>
<p>关机后进入<strong>编辑虚拟机设置</strong>页，移出CD/DVD设备或是取消勾选该设备的启动时连接</p>
<p>完成后启动虚拟机进入安装好的系统，开机后使用安装时设置的用户密码登录，初次启动会有一些提示框，基本一直下一步即可进入桌面</p>
<div class="note info flat"><p>进入系统以后同样会遇到分辨率问题，可以参照上文调整分辨率和缩放</p>
</div>
<div class="note success flat"><p>一切准备好之后，就可以进行下一步的换源操作了</p>
</div><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="ubuntu_installation-3"><div class="note info flat"><p>此处以<a target="_blank" rel="noopener" href="https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/">清华源</a>为例进行配置，同样也可以选择企业提供的镜像源，如<a target="_blank" rel="noopener" href="https://developer.aliyun.com/mirror/ubuntu?spm=a2c6h.13651102.0.0.4fdf1b11zOHAIu">阿里源</a>，在一定程度上企业源会比大学镜像源更加可靠，可以酌情选择</p>
</div>
<p>换源实质上就是更改系统所使用的软件源的配置文件，因此要做的事就只有两件：</p>
<ul>
<li>去使用的源的网站获取当前版本的配置文件</li>
<li>修改系统中原本的配置文件</li>
</ul>
<p><strong>清华源</strong>中给出的<strong>Ubuntu 20.04 LTS</strong>版本的配置文件如下，复制该配置文件内容</p>
<figure class="highlight txt"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br></pre></td><td class="code"><pre><span class="line"># 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释</span><br><span class="line">deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse</span><br><span class="line"># deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse</span><br><span class="line">deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse</span><br><span class="line"># deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse</span><br><span class="line">deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse</span><br><span class="line"># deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse</span><br><span class="line">deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse</span><br><span class="line"># deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse</span><br><span class="line"></span><br><span class="line"># 预发布软件源，不建议启用</span><br><span class="line"># deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse</span><br><span class="line"># deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-proposed main restricted universe multiverse</span><br></pre></td></tr></table></figure>
<div class="note warning flat"><p>对于不同版本的Ubuntu，配置文件会有所不同，体现在链接后面的<code>focal</code>字样，该字段是Ubuntu的版本号，例如Ubuntu 18.04的版本号就是仿生鱼<code>bionic</code>而Ubuntu 22.04LTS版本号为<code>jammy</code></p>
<p>如果选择了错误的配置文件，会使得后续<code>apt update</code>环节出现奇怪的错误，因此<strong>一定要选择版本对应的配置文件</strong></p>
</div>
<p>在终端中输入</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">sudo vim /etc/apt/sources.list</span><br></pre></td></tr></table></figure>
<div class="note info flat"><p>如果还没有安装vim，可以</p>
<ul>
<li>改成使用vi而不是vim，在换源后再安装vim</li>
<li>执行<code>sudo apt-get update &amp;&amp; sudo apt-get install vim</code>安装vim继续</li>
</ul>
</div>
<p>首先注释掉原先的内容，然后在文件尾追加上文中复制的配置文件，之后保存退出</p>
<div class="note orange icon-padding flat"><i class="note-icon fas fa-lightbulb"></i><p>在vim中使用<code>I</code>进入编辑模式，使用<code>ESC</code>退出编辑模式</p>
<p>在非编辑模式下输入<code>:wq</code>保存并退出，输入<code>:q</code>退出，输入<code>:q!</code>放弃更改退出</p>
</div>
<div class="note orange icon-padding flat"><i class="note-icon fas fa-lightbulb"></i><p>在vim中可以使用如下方式批量添加注释</p>
<ol>
<li><code>Ctrl</code>+<code>V</code>进入列选择模式</li>
<li><code>J</code> / <code>K</code> / <code>↓</code> / <code>↑</code> 移动光标选中行</li>
<li><code>Shift</code>+<code>I</code>（这一步会让光标移到选中列的第一行第一个字符之前，但实际上光标在所有选中行的第一个字符之前）</li>
<li><code>Shift</code>+<code>3</code>（输入注释符<code>#</code>）</li>
<li><code>ESC</code>退出</li>
</ol>
</div>
<p>保存了新的配置文件以后，执行如下命令执行更新</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">sudo apt-get update</span><br></pre></td></tr></table></figure>
<div class="note success flat"><p>执行了更新之后就完成了换源的全部步骤，可以进入下一步安装其他软件环境了！</p>
</div><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="ubuntu_installation-4"><div class="note primary flat"><p>安装C/C++环境</p>
</div>
<p>终端执行以下命令</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">sudo apt install binutils</span><br><span class="line">sudo apt install gcc</span><br></pre></td></tr></table></figure>
<p>或者使用一行命令</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">sudo apt install binutils &amp;&amp; sudo apt install gcc</span><br></pre></td></tr></table></figure>
<p>检查是否安装成功</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">gcc -v</span><br></pre></td></tr></table></figure>
<p>如果输出<code>gcc</code>的版本号则表明安装成功</p>
<div class="note primary flat"><p>安装NASM</p>
</div>
<p>首先检查已经安装的<code>nasm</code>版本</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">nasm -v</span><br></pre></td></tr></table></figure>
<p>如果已经是<code>2.15^</code>版本则可以跳过本节，如果不是的话则需要安装<code>2.15^</code>版本的<code>nasm</code></p>
<p>首先根据<a target="_blank" rel="noopener" href="https://www.linuxfromscratch.org/blfs/view/svn/general/nasm.html">网上教程</a>的指引，下载压缩包</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">$ wget https://www.nasm.us/pub/nasm/releasebuilds/2.15.05/nasm-2.15.05.tar.xz</span><br><span class="line">Download time (Average speed) - `nasm-2.15.05.tar.xz` saved [995732/995732]</span><br></pre></td></tr></table></figure>
<p>然后进行解压和配置文件生成</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br></pre></td><td class="code"><pre><span class="line">$ tar -xvf nasm-2.15.05.tar.xz</span><br><span class="line">nasm-2.15.05/</span><br><span class="line">nasm-2.15.05/AUTHORS</span><br><span class="line">...</span><br><span class="line">nasm-2.15.05/doc/nasmlogw.png</span><br><span class="line"></span><br><span class="line">$ <span class="built_in">cd</span> nasm-2.15.05/</span><br><span class="line"></span><br><span class="line">$ ./configure</span><br><span class="line">checking <span class="keyword">for</span> prefix by checking <span class="keyword">for</span> nasm... no</span><br><span class="line">...</span><br><span class="line">configure: creating ./config.status</span><br><span class="line">config.status: creating Makefile</span><br><span class="line">config.status: creating doc/Makefile</span><br><span class="line">config.status: creating config/config.h</span><br></pre></td></tr></table></figure>
<div class="note warning flat"><p>上面的bash代码不要直接复制，其包含了输出的显示</p>
<p>只有 <strong>$</strong> 后的内容才需要输入，其他的内容为预期的输出显示</p>
</div>
<p>最后进行<code>make</code>安装</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">make</span><br><span class="line">sudo make install</span><br></pre></td></tr></table></figure>
<div class="note primary flat"><p>安装LLVM</p>
</div>
<p>如果打算使用<code>LLVM</code>作为编译器，则可以参照<a target="_blank" rel="noopener" href="https://apt.llvm.org/">官网安装指南</a>进行安装，下面给出脚本安装方式</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">sudo bash -c <span class="string">&quot;<span class="subst">$(wget -O - https://apt.llvm.org/llvm.sh)</span>&quot;</span></span><br></pre></td></tr></table></figure>
<div class="note info flat"><p>安装后的clang可能不叫作clang，因此直接执行<code>clang -v</code>可能会报错未安装</p>
<p>输入<code>clang</code>后按<code>TAB</code>两次可以看到安装了的<code>clang</code>版本，例如<code>clang-14</code>和<code>clang++-14</code>，这样输入<code>clang-14 -v</code>就能够显示版本了</p>
<p>之后在编写makefile的时候也要注意不要将编译器写为单独的clang，而是带后缀的安装了的版本</p>
</div>
<div class="note primary flat"><p>安装其他工具</p>
</div>
<p>终端执行以下命令</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br></pre></td><td class="code"><pre><span class="line">sudo apt install git</span><br><span class="line">sudo apt install qemu</span><br><span class="line">sudo apt install cmake</span><br><span class="line">sudo apt install libncurses5-dev</span><br><span class="line">sudo apt install bison</span><br><span class="line">sudo apt install flex</span><br><span class="line">sudo apt install libssl-dev</span><br><span class="line">sudo apt install libc6-dev-i386</span><br><span class="line">sudo apt install gcc-multilib </span><br><span class="line">sudo apt install g++-multilib</span><br></pre></td></tr></table></figure>
<p>或是输入一行指令</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">sudo apt install git &amp;&amp; sudo apt install qemu &amp;&amp; sudo apt install cmake &amp;&amp; sudo apt install libncurses5-dev &amp;&amp; sudo apt install bison &amp;&amp; sudo apt install flex &amp;&amp; sudo apt install libssl-dev &amp;&amp; sudo apt install libc6-dev-i386 &amp;&amp; sudo apt install gcc-multilib &amp;&amp; sudo apt install g++-multilib</span><br></pre></td></tr></table></figure><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="ubuntu_installation-5"><div class="note info flat"><p>通过设置Ubuntu环境下的SSH，可以让Windows能够通过SSH连接到Ubuntu环境的shell，从而可以在Windows环境下完成大部分操作</p>
<p>除此之外，通过VSCode中的Remote插件，可以在Windows环境下编写Ubuntu中的文件</p>
<p>如果你希望在Ubuntu中完成所有代码的编写，则这一步不是必须的</p>
</div>
<p>通过如下命令安装<code>ssh</code></p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">sudo apt-get install ssh</span><br></pre></td></tr></table></figure>
<p>之后配置密钥登录所必须的文件目录和文件并设置权限</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br></pre></td><td class="code"><pre><span class="line"><span class="built_in">mkdir</span> ~/.ssh &amp;&amp; <span class="built_in">cd</span> ~/.ssh</span><br><span class="line"><span class="built_in">touch</span> authorized_keys</span><br><span class="line"><span class="built_in">chmod</span> 600 ~/.ssh/authorized_keys</span><br><span class="line"><span class="built_in">chmod</span> 700 ~/.ssh</span><br></pre></td></tr></table></figure>
<p>通过修改<code>sshd</code>配置文件关闭密码登录并打开密钥登录</p>
<p>打开<code>sshd_config</code>文件</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">sudo vim /etc/ssh/sshd_config</span><br></pre></td></tr></table></figure>
<p>之后在<code># Authentication:</code>下修改如下键值，同时如果有注释则关闭注释</p>
<figure class="highlight txt"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">PubkeyAuthentication yes</span><br><span class="line">PasswordAuthentication no</span><br></pre></td></tr></table></figure>
<p>保存退出后重启<code>sshd</code>服务</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">service sshd restart</span><br></pre></td></tr></table></figure>
<p>之后，需要在<code>VMWare Workstation</code>里转发<code>22</code>端口到虚拟机</p>
<p>为了完成这个任务，需要先获取虚拟机的<code>ip</code>地址</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br></pre></td><td class="code"><pre><span class="line">$ ip a</span><br><span class="line">...</span><br><span class="line">inet 192.168.XXX.XXX/24 brd 192.168.XXX.255 scope global dynamic noprefixroute ens33</span><br><span class="line">...</span><br></pre></td></tr></table></figure>
<p>可以看到输出中给出的<code>ipv4</code>地址，这里假设为<code>192.168.85.130</code>，记住该地址</p>
<p>进入<code>VMWare Workstation</code>中的 <strong>编辑-虚拟网络编辑器-更改设置-VMnet8 NAT模式-NAT设置</strong>，添加端口转发规则</p>
<table>
<thead>
<tr>
<th style="text-align:center">主机端口</th>
<th style="text-align:center">类型</th>
<th style="text-align:center">虚拟机IP地址</th>
<th style="text-align:center">虚拟机端口</th>
<th style="text-align:center">描述</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center">22</td>
<td style="text-align:center">TCP</td>
<td style="text-align:center">192.168.85.130</td>
<td style="text-align:center">22</td>
<td style="text-align:center">SSH</td>
</tr>
</tbody>
</table>
<p>保存并应用设置就完成了端口转发的添加</p>
<p>最后，为了能够让<code>Windows</code>能够顺利连接上虚拟机，需要在<code>Windows</code>环境下生成密钥并且拷贝公钥到上文中创建的<code>authorized_keys</code>文件中</p>
<div class="note warning flat"><p>下述操作将在Windows环境中进行</p>
</div>
<p>首先，需要在<code>Windows</code>环境下安装<code>OpenSSH</code>，参照<a target="_blank" rel="noopener" href="https://docs.microsoft.com/zh-cn/windows-server/administration/openssh/openssh_install_firstuse">Microsoft官方文档</a>给出的指南，可以使用<code>Powershell</code>安装<code>OpenSSH</code></p>
<p>以管理员身份运行<code>Powershell</code>并且检查<code>OpenSSH</code>的安装情况</p>
<figure class="highlight powershell"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="built_in">Get-WindowsCapability</span> <span class="literal">-Online</span> | <span class="built_in">Where-Object</span> Name <span class="operator">-like</span> <span class="string">&#x27;OpenSSH*&#x27;</span></span><br></pre></td></tr></table></figure>
<p>如果两者均没有安装，则会返回如下输出</p>
<figure class="highlight txt"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br></pre></td><td class="code"><pre><span class="line">Name  : OpenSSH.Client~~~~0.0.1.0</span><br><span class="line">State : NotPresent</span><br><span class="line"></span><br><span class="line">Name  : OpenSSH.Server~~~~0.0.1.0</span><br><span class="line">State : NotPresent</span><br></pre></td></tr></table></figure>
<p>之后，根据需要安装对应未安装的组件</p>
<figure class="highlight powershell"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment"># Install the OpenSSH Client</span></span><br><span class="line"><span class="built_in">Add-WindowsCapability</span> <span class="literal">-Online</span> <span class="literal">-Name</span> OpenSSH.Client~~~~<span class="number">0.0</span>.<span class="number">1.0</span></span><br><span class="line"></span><br><span class="line"><span class="comment"># Install the OpenSSH Server</span></span><br><span class="line"><span class="built_in">Add-WindowsCapability</span> <span class="literal">-Online</span> <span class="literal">-Name</span> OpenSSH.Server~~~~<span class="number">0.0</span>.<span class="number">1.0</span></span><br></pre></td></tr></table></figure>
<p>这两者都应该返回如下输出</p>
<figure class="highlight txt"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line">Path          :</span><br><span class="line">Online        : True</span><br><span class="line">RestartNeeded : False</span><br></pre></td></tr></table></figure>
<p>之后，使用终端生成密钥</p>
<figure class="highlight powershell"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br></pre></td><td class="code"><pre><span class="line">ssh<span class="literal">-keygen</span> <span class="literal">-t</span> rsa</span><br><span class="line"><span class="comment"># Or</span></span><br><span class="line">ssh<span class="literal">-keygen</span> <span class="literal">-t</span> ed25519</span><br></pre></td></tr></table></figure>
<p>根据提示设置对应的文件位置（默认为<code>%USERPROFILE%\.ssh\id_rsa</code>）以及密码</p>
<p>然后使用<code>cat</code>命令显示公钥内容，假定公钥文件为<code>C:\Users\Linloir\.ssh\id_ed25519.pub</code></p>
<figure class="highlight powershell"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line"><span class="built_in">cat</span> C:\Users\Linloir\.ssh\id_ed25519.pub</span><br></pre></td></tr></table></figure>
<p>会返回一个类似如下的输出</p>
<figure class="highlight txt"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">ssh-ed25519 AAAAC3N******************TzC6 linloir@******</span><br></pre></td></tr></table></figure>
<p>拷贝该输出，并回到<code>Linux</code>操作</p>
<div class="note orange icon-padding flat"><i class="note-icon fas fa-lightbulb"></i><p>如果希望在Windows和Linux虚拟机间复制内容，可以在VMWare中安装VMWare Tools或Open VM Tools</p>
</div>
<div class="note orange icon-padding flat"><i class="note-icon fas fa-lightbulb"></i><p>如果VMWare Tools安装选项为灰色，可以参照<a target="_blank" rel="noopener" href="https://blog.csdn.net/qq_40259641/article/details/79022844">网上的博客</a>进行如下操作</p>
<ol>
<li>关闭虚拟机</li>
<li>打开虚拟机设置</li>
<li>如果在前序步骤删除了CD/DVD设备，则添加回来</li>
<li>取消勾选CD/DVD设备的 <em>启动时连接</em>，同时勾选 <em>使用物理驱动器-自动检测</em></li>
<li>重启虚拟机</li>
</ol>
</div>
<div class="note info flat"><p>在测试过程中，安装VMWare Workstation自带的VMWare Tools会提示建议安装Open VM Tools</p>
<p>可以结束安装并参照<a target="_blank" rel="noopener" href="https://docs.vmware.com/en/VMware-Tools/12.0.0/com.vmware.vsphere.vmwaretools.doc/GUID-C48E1F14-240D-4DD1-8D4C-25B6EBE4BB0F.html">官方文档</a>或是<a target="_blank" rel="noopener" href="https://github.com/vmware/open-vm-tools">Github页面</a>中的指南使用<code>sudo apt-get install open-vm-tools-desktop</code>安装</p>
</div>
<p>在<code>Linux</code>终端中使用<code>vim</code>修改<code>authorized_keys</code>文件</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">vim ~/.ssh/authorized_keys</span><br></pre></td></tr></table></figure>
<p>进入编辑模式粘贴上文中复制的密钥内容后保存退出</p>
<p>最后可以在<code>Windows</code>环境下使用下述命令测试连接到虚拟机</p>
<div class="note info flat"><p>在<code>&lt;username&gt;</code>中填写<code>Linux</code>用户名，<code>&lt;ip&gt;</code>填写前文中获得的虚拟机<code>ip</code>地址</p>
</div>
<figure class="highlight powershell"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">ssh &lt;username&gt;<span class="selector-tag">@</span>&lt;ip&gt;</span><br></pre></td></tr></table></figure>
<div class="note success flat"><p>如果连接成功，则证明完成了SSH的全部配置内容，同样，也完成了Ubuntu环境的全部配置！</p>
<p>接下来，可以直接进入下一节或是跟随教程完成Windows下使用工具的配置和安装</p>
</div><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div></div></div>
<h3 id="Windows环境配置指南">Windows环境配置指南</h3>
<div class="note info flat"><p>如果希望在Linux环境下完成所有代码编写与调试，可以跳过此节</p>
</div>
<p>为了提升代码编写体验，在本节中将会介绍如何在Windows环境下配置和使用VSCode以及其插件，从而能够在Windows环境下使用VSCode编写Linux中的代码</p>
<p>同样，还会介绍通过Windows终端连接Linux终端的方式，并在Windows环境下配置Linux环境中的ZSH终端以及powerlevel10k主题</p>
<div class="tabs" id="windows_environment"><ul class="nav-tabs"><button type="button" data-href="#windows_environment-1" class="tab active">安装VSCode与插件</button><button type="button" data-href="#windows_environment-2" class="tab">好看的终端</button></ul><div class="tab-contents"><div class="tab-item-content active" id="windows_environment-1"><p>前往<a target="_blank" rel="noopener" href="https://code.visualstudio.com/">VSCode官网</a>下载并安装VSCode</p>
<p>安装完成后进入VSCode页面，在左侧栏中有一积木样图标，点击进入VSCode的插件安装页</p>
<p>搜索并安装以下插件，具体安装的插件可以根据自己的喜好选择，其中<code>Remote</code>家族为连接虚拟机所需要的插件</p>
<ul>
<li>C/C++ (Microsoft)</li>
<li>Chinese (Simplified)(简体中文) Language Pack For Visual Studio Code (Microsoft)</li>
<li>CMake Tools (Microsoft)</li>
<li>Hex Editor (Microsoft)</li>
<li>Remote - SSH (Microsoft)</li>
<li>Remote - SSH:Editing Configuration Files (Microsoft)</li>
<li>The Netwide Assembler (NASM) (Right)</li>
</ul>
<p>在完成了<code>Remote</code>家族插件的安装后，侧边栏会有一远程连接的图标，点击可以进入远程资源管理器</p>
<p>选择<code>SSH Target</code>以添加虚拟机，之后在<code>SSH TARGETS</code>菜单处点击添加键，在弹出窗口中输入</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">ssh &lt;username&gt;@&lt;ip&gt;</span><br></pre></td></tr></table></figure>
<p>在弹出的<code>Select SSH configuration file to update</code>页面中选择默认选项，也即直接按回车</p>
<p>之后在左侧<code>SSH TARGETS</code>菜单栏能够看到新添加的地址，鼠标移至其上方，点击其右侧文件夹图标在新窗口中打开</p>
<p>在<code>Select the platform for the remote host</code>窗口中选择<code>Linux</code></p>
<p>之后输入在上文 <em>配置SSH</em> 中生成密钥步骤中设置的密码即完成连接</p>
<div class="note success flat"><p>快试试在侧边栏中的文件页面中打开Ubuntu中的项目文件夹吧！</p>
</div><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div><div class="tab-item-content" id="windows_environment-2"><div class="note info flat"><p>在我的实验中，在Windows中使用了<code>Windows Terminal</code>，在Linux中使用了<code>ZSH</code>+<code>powerlevel10k</code>的组合，你也可以根据你的喜好设置自己的终端和主题</p>
</div>
<p>首先，在Windows Terminal中使用ssh连接到虚拟机终端</p>
<p>之后，根据<a target="_blank" rel="noopener" href="https://github.com/ohmyzsh/ohmyzsh/wiki/Installing-ZSH">ZSH Wiki页面</a>的教程安装ZSH</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br></pre></td><td class="code"><pre><span class="line"><span class="comment"># 安装</span></span><br><span class="line">$ sudo apt install zsh</span><br><span class="line"></span><br><span class="line"><span class="comment"># 验证安装</span></span><br><span class="line">$ zsh --version</span><br><span class="line">zsh 5.8 <span class="comment"># 或者是更新的版本</span></span><br><span class="line"></span><br><span class="line"><span class="comment"># 设置为默认shell</span></span><br><span class="line">$ chsh -s $(<span class="built_in">which</span> zsh)</span><br><span class="line"></span><br><span class="line"><span class="comment"># 退出登录</span></span><br><span class="line">$ <span class="built_in">exit</span></span><br></pre></td></tr></table></figure>
<p>然后重新使用ssh登录，初次进入会进行设置，可以根据自己的喜好进行设置或者直接选择<code>Populate your ~/.zshrc with the configuration recommended by the system administrator and exit</code>进行默认设置</p>
<div class="note info flat"><p>关于各个设置项作用将会在日后更新（如果有空）</p>
</div>
<p>之后依据<a target="_blank" rel="noopener" href="https://ohmyz.sh/#install">Oh My Zsh官网</a>来安装Oh-My-Zsh框架</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">sh -c <span class="string">&quot;<span class="subst">$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)</span>&quot;</span></span><br></pre></td></tr></table></figure>
<div class="note success flat"><p>完成设置就代表着Zsh安装完成，可以进行后续主题安装</p>
</div>
<p>在安装powerlevel10k主题前，需要安装其依赖的字体</p>
<ul>
<li><a target="_blank" rel="noopener" href="https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf">MesloLGS NF Regular.ttf</a></li>
<li><a target="_blank" rel="noopener" href="https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf">MesloLGS NF Bold.ttf</a></li>
<li><a target="_blank" rel="noopener" href="https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf">MesloLGS NF Italic.ttf</a></li>
<li><a target="_blank" rel="noopener" href="https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf">MesloLGS NF Bold Italic.ttf</a></li>
</ul>
<p>双击打开下载的字体文件进行安装，之后进入<code>Windows Terminal</code> → <code>设置</code> → <code>配置文件</code> → <code>默认值</code> → <code>外观</code> → <code>字体</code>，选择<code>MesloLGS NF</code>字体</p>
<p>然后使用<code>git</code>安装该主题</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">git <span class="built_in">clone</span> --depth=1 https://github.com/romkatv/powerlevel10k.git <span class="variable">$&#123;ZSH_CUSTOM:-<span class="variable">$HOME</span>/.oh-my-zsh/custom&#125;</span>/themes/powerlevel10k</span><br></pre></td></tr></table></figure>
<p>可以使用<code>gitee</code>加速下载</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br></pre></td><td class="code"><pre><span class="line">git <span class="built_in">clone</span> --depth=1 https://gitee.com/romkatv/powerlevel10k.git <span class="variable">$&#123;ZSH_CUSTOM:-<span class="variable">$HOME</span>/.oh-my-zsh/custom&#125;</span>/themes/powerlevel10k</span><br></pre></td></tr></table></figure>
<p>在<code>~/.zshrc</code>中设置<code>ZSH_THEME=&quot;powerlevel10k/powerlevel10k</code>并重启Zsh</p>
<figure class="highlight bash"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">vim ~/.zshrc</span><br><span class="line"><span class="built_in">exec</span> zsh</span><br></pre></td></tr></table></figure>
<p>之后根据设置向导设置主题即可得到一个漂亮的终端啦</p><button type="button" class="tab-to-top" aria-label="scroll to top"><i class="fas fa-arrow-up"></i></button></div></div></div>
<div class="note success flat"><p>目前为止，环境已经基本配置完成了<br>
接下来就让我们开始愉快的操作系统实验之旅吧！</p>
</div>

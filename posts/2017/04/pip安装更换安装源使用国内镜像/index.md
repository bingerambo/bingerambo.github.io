# PIP安装：更换安装源，使用国内镜像



<div class="post">
    <h1 class="postTitle">
            <a id="cb_post_title_url" class="postTitle2" href="#">PIP安装：更换安装源，使用国内镜像。</a>
        </h1>
    <div class="clear"></div>
    <div class="postBody">
        <div id="cnblogs_post_body">
            <p>对于Python开发用户来讲，经常使用PIP安装软件包。但是由于PIP默认安装源是在国外，经常出现下载后安装出错问题。所以把PIP安装源替换成国内镜像，可以大幅提升下载速度，还可以提高安装成功率。</p>
            <h2>国内源：</h2>
            <p>新版ubuntu要求使用https源，要注意。</p>
            <p>清华：https://pypi.tuna.tsinghua.edu.cn/simple</p>
            <p>阿里云：http://mirrors.aliyun.com/pypi/simple/</p>
            <p>中国科技大学 https://pypi.mirrors.ustc.edu.cn/simple/</p>
            <p>华中理工大学：http://pypi.hustunique.com/</p>
            <p>山东理工大学：http://pypi.sdutlinux.org/&nbsp;</p>
            <p>豆瓣：http://pypi.douban.com/simple/</p>
            <h2>临时使用：</h2>
            <p>可以在使用pip的时候加参数-i http://pypi.douban.com/simple/
                <br>
                <br>例如：pip install -i http://pypi.douban.com/simple/ django，这样就会从豆瓣这边的镜像去安装django库。
                <br>&nbsp;</p>
            <h2>永久修改，一劳永逸：</h2>
            <p>Linux下，修改 ~/.pip/pip.conf (没有就创建一个文件夹及文件。文件夹要加“.”，表示是隐藏文件夹)</p>
            <p>内容如下：</p>
            <div class="cnblogs_code">
                <pre>[<span style="color: #0000ff">global</span><span style="color: #000000">]
index</span>-url = https:<span style="color: #008000">//</span><span style="color: #008000">pypi.douban.com/simple/<br></span></pre>
                <pre>[install]
trusted-host=mirrors.aliyun.com</pre>
            </div>
            <p>windows下，直接在user目录中创建一个pip目录，如：C:\Users\xx\pip，新建文件pip.ini。内容同上。</p>
        </div>
    </div>
</div>

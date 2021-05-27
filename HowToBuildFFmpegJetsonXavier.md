---


---

<h1 id="ffmpeg-guide-for-jetson-nx-xavier">FFmpeg Guide for Jetson NX Xavier</h1>
<p>Normally it is enough to follow other guides such as <a href="https://github.com/jocover/jetson-ffmpeg">this</a> or <a href="https://github.com/daniilidis-group/ffmpeg_image_transport">this</a> to have ffmpeg compression operative in the Jetson we intend to use. These guides are effective, because by themselves they work, i.e., it is possible to compress the selected videos, however when it comes to integrate with ROS, there are problems with libraries and different elements of ffmpeg. In this guide we will install and compile ffmpeg to be used in a <strong>Jetson NX Xavier</strong> with <strong>Ubuntu 20.04 LTS</strong>.</p>
<p>First of all we have to uninstall the ffmpeg that <strong>Ubuntu 20.04</strong> comes with, since we will install later one that suits our needs.</p>
<p><code>sudo apt remove ffmpeg</code></p>
<p>Now, we will merge the two guides presented above. First, ffmpeg will be installed for the jetson:</p>
<pre><code>git clone https://github.com/jocover/jetson-ffmpeg.git
cd jetson-ffmpeg
mkdir build
cd build
cmake ..
make
sudo make install
sudo ldconfig
</code></pre>
<p>And now we will proceed to the installation of the different dependencies that ffmpeg needs. Let’s build ffmpeg from scratch.</p>
<pre><code>sudo apt-get update -qq &amp;&amp; sudo apt-get -y install \
autoconf \
automake \
build-essential \
cmake \
git-core \
libass-dev \
libfreetype6-dev \
libsdl2-dev \
libtool \
libva-dev \
libvdpau-dev \
libvorbis-dev \
libxcb1-dev \
libxcb-shm0-dev \
libxcb-xfixes0-dev \
pkg-config \
texinfo \
wget \
zlib1g-dev \
libx264-dev \
libx265-dev
</code></pre>
<p>For the moment, as you can see, we will follow this <a href="https://github.com/daniilidis-group/ffmpeg_image_transport/blob/master/docs/compile_ffmpeg.md">guide</a>. Make your own ffmpeg directory. Go back to your home directory:</p>
<pre><code>ffmpeg_dir=&lt;absolute_path_to_empty_directory_for_ffmpeg_stuff&gt;
mkdir -p $ffmpeg_dir/build $ffmpeg_dir/bin
</code></pre>
<p>Build <strong>yasm</strong>:</p>
<pre><code>cd $ffmpeg_dir
wget -O yasm-1.3.0.tar.gz https://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz &amp;&amp; \
tar xzvf yasm-1.3.0.tar.gz &amp;&amp; \
cd yasm-1.3.0 &amp;&amp; \
./configure --prefix="$ffmpeg_dir/build" --bindir="$ffmpeg_dir/bin" &amp;&amp; \
make &amp;&amp; \
make install
</code></pre>
<p>Build <strong>nasm</strong>:</p>
<pre><code>cd $ffmpeg_dir
wget https://www.nasm.us/pub/nasm/releasebuilds/2.14.02/nasm-2.14.02.tar.bz2 &amp;&amp; \
tar xjvf nasm-2.14.02.tar.bz2 &amp;&amp; \
cd nasm-2.14.02 &amp;&amp; \
./autogen.sh &amp;&amp; \
PATH="${ffmpeg_dir}/bin:$PATH" ./configure --prefix="${ffmpeg_dir}/build" --bindir="${ffmpeg_dir}/bin" &amp;&amp; \
make &amp;&amp; \
make install
</code></pre>
<blockquote>
<p><strong>Note</strong>: It is possible that both nasm and yasm give problems when configuring and installing them, in my case, nasm gave me the problem, since I was using a previous version to the one I should use. Therefore I recommend before any error to check that the versions used are the correct ones.</p>
</blockquote>
<p>Get <strong>nvcodec</strong> headers:</p>
<pre><code>cd $ffmpeg_dir
git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git
cd nv-codec-headers

</code></pre>
<p>At this point you will have to pick the right version of the headers. You need the one that matches your driver. For instance:</p>
<pre><code>git checkout n8.2.15.8

</code></pre>
<p>will get you a version that works with with Linux 410.48 or newer. If you mess up here, you will later get an error that looks like this:</p>
<pre><code>[hevc_nvenc @ 0x7fc67c03a580] Cannot load libnvidia-encode.so.1
[hevc_nvenc @ 0x7fc67c03a580] The minimum required Nvidia driver for nvenc is 418.30 or newer

</code></pre>
<p>Now create a modified Makefile that points to the right location (in the nv-codev-headers folder inside of ffmpeg main folder):</p>
<pre><code>tmp_var="${ffmpeg_dir//"/"/"\/"}"
sed "s/\/usr\/local/${tmp_var}\/build/g" Makefile &gt; Makefile.tmp
make -f Makefile.tmp install
</code></pre>
<p>And now we jump to the another <a href="https://github.com/jocover/jetson-ffmpeg">guide</a>, because we need a ffmpeg patched for being used in a <strong>Jetson</strong>. Inside of the <strong>ffmpeg folder</strong> that we have created:</p>
<pre><code>git clone git://source.ffmpeg.org/ffmpeg.git -b release/4.2 --depth=1
cd ffmpeg
wget https://github.com/jocover/jetson-ffmpeg/raw/master/ffmpeg_nvmpi.patch
git apply ffmpeg_nvmpi.patch
PATH="$ffmpeg_dir/bin:$PATH" PKG_CONFIG_PATH="$ffmpeg_dir/build/lib/pkgconfig" ./configure --prefix=${ffmpeg_dir}/build --extra-cflags=-I${ffmpeg_dir}/build/include --extra-ldflags=-L${ffmpeg_dir}/build/lib --bindir=${ffmpeg_dir}/bin --enable-gpl --enable-libx264 --enable-libx265 --enable-nonfree --enable-shared --enable-nvmpi
make
</code></pre>
<p>Now build and install (runs for a few minutes!):</p>
<pre><code>PATH="$ffmpeg_dir/bin:${PATH}:/usr/local/cuda/bin" make &amp;&amp; sudo make install &amp;&amp; hash -r
</code></pre>
<p>At this point we just have to test if we can use ffmpeg properly, so if you want you can download and test the compression:</p>
<pre><code>ffmpeg -i input_file -c:v h264_nvmpi &lt;output.mp4&gt;
</code></pre>
<h2 id="its-time-for-ros">It’s time for ROS</h2>
<p>In this section we will follow the instructions of the following <a href="https://github.com/daniilidis-group/ffmpeg_image_transport/blob/master/docs/compile_ffmpeg.md">repository</a>.</p>
<h4 id="downloading">Downloading</h4>
<p>Create a catkin workspace (if not already there) and download the ffmpeg image transport and other required packages:</p>
<pre><code>mkdir -p catkin_ws/src
cd catkin_ws/src
git clone https://github.com/daniilidis-group/ffmpeg_image_transport_msgs.git
git clone https://github.com/daniilidis-group/ffmpeg_image_transport.git
</code></pre>
<p>We have compiled and built ffmpeg from scratch so we have to follow the next instructions:</p>
<pre><code>ffmpeg_dir=&lt;here the ffmpeg_dir&gt;
</code></pre>
<p>We have already set <code>ffmpeg_dir</code> variable .</p>
<pre><code>catkin config -DCMAKE_BUILD_TYPE=Release -DFFMPEG_LIB=${ffmpeg_dir}/build/lib -DFFMPEG_INC=${ffmpeg_dir}/build/include
</code></pre>
<h4 id="compiling">Compiling</h4>
<p>This should be easy as running the following inside your catkin workspace:</p>
<pre><code>catkin config -DCMAKE_BUILD_TYPE=Release -DFFMPEG_LIB=${ffmpeg_dir}/build/lib -DFFMPEG_INC=${ffmpeg_dir}/build/include
</code></pre>
<p>then compile should be as easy as this:</p>
<pre><code>catkin build ffmpeg_image_transport
</code></pre>
<h3 id="troubleshooting">Troubleshooting</h3>
<p>If the build fails, make sure you start from scratch:</p>
<pre><code>catkin clean ffmpeg_image_transport
</code></pre>


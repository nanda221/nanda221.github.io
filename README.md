# nanda221.github.io
Personal Blog Repo

# Docker容器启动流程

1. 基于[docker-libs](https://github.com/nanda221/docker-libs)apps/jekyll创建镜像；
2. 使用命令**docker build -t jekyll:0.3 .**构建镜像
3. 启动容器实例并挂载外部文件夹**docker run -d -p 8080:80 -p 10112:22 -v /Users/nanda221/opensrc/nanda221.github.io:/webapp jekyll:0.3**
4. 远程连接**ssh root@0.0.0.0 -p 10112**

所有的的安装过程都已经打到了/root/app.sh中，包括bundle install和bundle exec jekyll serve等。jekyll serve必须用bundle exec执行，这样才可以使用通过bundle安装的项目环境的gem依赖包。
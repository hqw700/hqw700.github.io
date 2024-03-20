## node install
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install stable
npm install -g hexo-cli

# hexo blog 源文件
+ hexo g // 生成
+ hexo s // 本地预览
+ hexo d // 发布

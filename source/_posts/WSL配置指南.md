---
title: WSL配置指南
date: 2020-04-20 18:45:29
---

1. 以管理员启动控制台，执行`Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux`
2. 在应用商店安装`Ubuntu 18.04`和`Windows Terminal`，配置`Ubuntu 18.04`为默认终端
3. 打开`Windows Terminal`控制台连接`Ubuntu 18.04`，输入用户名和密码，执行`sudo passwd root`设置管理员密码

<!-- more -->

4. 修改`/etc/sudoers`使sudo不再需要密码

  ```bash
  sudo chmod u+w /etc/sudoers
  sudo vim /etc/sudoers
  
  # - %sudo   ALL=(ALL:ALL) ALL
  # + %sudo   ALL=(ALL:ALL) NOPASSWD:ALL

  sudo chmod u-w /etc/sudoers
  ```

5. 更换`apt`源，清华源地址：<https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/>
6. 执行`sudo apt update`和`sudo apt upgrade -y`（此过程较为耗时，耐心等待）
7. 安装`oh-my-zsh`

  ```bash
  # 安装zsh
  sudo apt install zsh -y

  # 配置zsh
  chsh -s /bin/zsh

  # 确认配置成功
  echo $SHELL

  # 重新打开终端以生效，选择2

  # 由于网络的原因，在浏览器打开脚本 https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh 复制
  # 在终端中创建一个.sh文件，不要使用.txt修改后缀的方式，不然脚本执行有问题
  touch oh-my-zsh.sh
  chmod u+x oh-my-zsh.sh
  sh oh-my-zsh.sh
  ```

8. 安装`oh-my-zsh`相关插件

   1. [autojump](https://github.com/wting/autojump)

      ```bash
      git clone git://github.com/wting/autojump.git
      # 如果没有python
      # sudo apt install python -y
      cd autojump && ./install.py
      ```

      接着拷贝以下代码到 ~/.zshrc 末尾

      ```text
      [[ -s /home/wenjun/.autojump/etc/profile.d/autojump.sh ]] && source /home/wenjun/.autojump/etc/profile.d/autojump.sh
      autoload -U compinit && compinit -u
      ```

   2. [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions)

      ```bash
      git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
      ```

      在 ~/.zshrc 中添加

      ```text
      plugins=(
        ...
        zsh-autosuggestions
      )
      ```

   3. [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting)

      ```bash
      git clone git://github.com/zsh-users/zsh-syntax-highlighting $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
      ```

      在 ~/.zshrc 中添加

      ```text
      plugins=(
        ...
        zsh-syntax-highlighting
      )
      ```

   4. 执行`compaudit | xargs chmod g-w,o-w`修复不安全的脚本执行

9. 安装node及npm

    ```bash
    # 安装node
    sudo apt install nodejs -y

    # 安装npm
    sudo apt install npm -y

    # 配置淘宝源
    sudo npm config set registry https://registry.npm.taobao.org

    # 修改权限
    echo $(npm config get prefix)
    sudo chown -R $(whoami) <path>

    # 升级npm到最新
    sudo npm install -g npm@latest

    # 安装n模块
    sudo npm install -g n

    # 清空缓存
    sudo npm cache clean -f

    # 升级node为稳定版
    sudo n stable
    ```

10. 执行`ssh-keygen -o`以生成ssh密钥，`cat ~/.ssh/id_rsa.pub`查看公钥，在`GitLab`中添加公钥以访问代码库
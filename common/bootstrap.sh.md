# zsh

```
#!/bin/bash

unameOut="$(uname -s)"
case "${unameOut}" in
Linux*) machine=Linux ;;
Darwin*) machine=Mac ;;
CYGWIN*) machine=Cygwin ;;
MINGW*) machine=MinGw ;;
*) machine="UNKNOWN:${unameOut}" ;;
esac
echo ${machine}

# color
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m' # No Color

red() {
	printf "${RED}$@${NC}\n"
}

green() {
	printf "${GREEN}$@${NC}\n"
}

yellow() {
	printf "${YELLOW}$@${NC}\n"
}

echo $(green "==================Install ZShell================")
echo $(yellow "switch to zsh")
if test $machine="Linux"; then
	sudo apt install zsh
	echo $(red "Installed ZSH!Follow steps below to set zsh as your default shell client!")
	echo $(red "1. run 'sudo vim /etc/passwd'")
	echo $(red "2. find your name and change '/bin/bash' or something else into '/bin/zsh'!")
elif test $machine="Mac"; then
	echo $(yellow "MacOS: install homebrew")
	/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
	brew install zsh
	chsh -s /bin/zsh
fi
```

# app
```

#!/bin/zsh

unameOut="$(uname -s)"
case "${unameOut}" in
Linux*) machine=Linux ;;
Darwin*) machine=Mac ;;
CYGWIN*) machine=Cygwin ;;
MINGW*) machine=MinGw ;;
*) machine="UNKNOWN:${unameOut}" ;;
esac
echo ${machine}

# color
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m' # No Color

red() {
    printf "${RED}$@${NC}\n"
}

green() {
    printf "${GREEN}$@${NC}\n"
}

yellow() {
    printf "${YELLOW}$@${NC}\n"
}

# set proxy
set_proxy="export http_proxy=10.20.47.147:3128 https_proxy=10.20.47.147:3128 no_proxy=code.byted.org"
unset_proxy="unset http_proxy && unset https_proxy"

if test $machine = "Linux"; then
    echo $set_proxy
fi

# create vimrc
echo $(green "==========Set Environment==========")
echo $(yellow "Set basic vim settings into .vimrc")
FILE=~/.vimrc
if test -f "$FILE"; then
    echo $(yellow ".vimrc already exist!")
else
    touch $FILE
    chmod 755 $FILE
fi

echo $(green "Write .vimrc")
# get file and write content
cat >$FILE <<EOL
" keybindings
inoremap jk <Esc>
inoremap <C-k> <Up>
inoremap <C-j> <Down>
inoremap <C-h> <Left>
inoremap <C-l> <Right>

" normal settings
set tabstop=2
set number
syntax on
set path+=**
set wildmode=list
EOL

# setup ssh
echo $(yellow "Whether ssh-key has been generated?")
if ! [ -e "$HOME/.ssh/id_rsa.pub" ]; then
    echo "Generating ssh key..."
    echo "Please enter the email you want to associate with your ssh key: \c"
    read email
    ssh-keygen -t rsa -C "$email"
fi
echo $(green "Copy your ssh-key to gitlab setting")
echo $(cat $HOME/.ssh/id_rsa.pub)

# setup global git config
existEmail=$(git config --global user.email)
if test $existEmail != ""; then
    echo $(yellow "You have already set your global git email account.")
elif; then
    echo "Please enter the email you want to associcate in your every git commit account: \c"
    read email
    git config --global user.email "$email"
    echo "Please enter the name you want to display in your every git commit:\c"
    read name
    git config --global user.name "$name"
fi

# install some tools
echo $(green "==========Install tools==========")
if test $machine = "Linux"; then
    if test $+commands[autojump] = 0; then
        sudo apt-get install autojump

        autojumpSet=". /usr/share/autojump/autojump.sh"
        printf $autojumpSet >>$HOME/.zshrc
    fi

    sudo apt-get install -y htop
    sudo apt-get install -y python3-dev
elif test $machine = "Mac"; then
    # command line tools
    brew install autojump
    brew install zsh-autosuggestions
    echo $(red "add ' source $(brew --prefix)/share/zsh-autosuggestions/zsh-autosuggestions.zsh' to your ~/.zshrc")
    brew install zsh-syntax-highlighting
    echo $(red "add 'source $(brew --prefix)/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh' to your ~/.zshrc")
    brew install htop
    # Apps
    brew install --cask visual-studio-code
    brew install --cask iterm2
    brew install --cask postman
    brew install --cask typora
    brew install --cask github
    # install fira code
    brew tap homebrew/cask-fonts
    brew install --cask font-fira-code
    echo $(yellow "You can replace your code font-size with Fira Code in VSCode!")
fi

echo $(green "=========Initialize Node Environement========")
# check nvm installed!
if [ ! -d "$HOME/.nvm" ]; then
    if test $machine = "Linux"; then
        sudo curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
    elif test $machine = "Mac"; then
        brew install nvm
    fi
else
    echo $(yellow "You have already installed nvm!")
fi
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"                   # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion" # This loads nvm bash_completion
# install node v12
nvm install v12
npm install -g yarn yrm commitizen cz-conventional-changelog
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc

echo $(green "==========Install OH_MY_ZSH==========")
if ! [ -e "$HOME/.oh-my-zsh/oh-my-zsh.sh" ]; then
    echo $(yellow "Install oh-my-zsh")
    rm -rf ~/.oh-my-zsh
    sh -c "$(curl --progress-bar -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
elif; then
    echo $(yellow "You have installed oh-my-zsh")
fi

# install plugins for oh-my-zsh
# syntx highlight
if ! [ -e "$HOME/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting/README.md" ]; then
    git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
    echo $(red "To activate zsh-syntax-highlight, add it to oh-my-zsh plugins which in ~/.zshrc")
    echo $(red "plugins=( [plugins...] zsh-syntax-highlighting)")
fi
# autosuggestion
if ! [ -e "$HOME/.oh-my-zsh/custom/plugins/zsh-autosuggestions/README.md" ]; then
    git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
    echo $(red "To activate zsh-autosuggestions, add it to oh-my-zsh plugins which in ~/.zshrc")
    echo $(red "plugins=( [plugins...] zsh-autosuggestions)")
fi

echo $unset_proxy
echo $(green "================Finished!================")
echo $(green "Final Step! Run command below to active changes!")
echo $(yellow "source ~/.zshrc")
```

# Maintainer: frantic1048 <i@frantic1048.com>
# Contributor: Filipe Laíns (FFY00) <lains@archlinux.org>
# Contributor: Michael Hansen <zrax0111 gmail com>
# Contributor: Francisco Magalhães <franmagneto gmail com>

pkgname=code-transparent
_pkgname=code
pkgdesc='The Open Source build of Visual Studio Code (vscode) editor - with transparency enabled.'
# Important: Remember to check https://github.com/microsoft/vscode/wiki/How-to-Contribute#prerequisites for target node version
# NodeJS versioning cheatsheet:
#   - carbon: 8
#   - dubnium: 10
#   - erbium: 12
#   - fermium: 14
# Important: Remember to check https://github.com/microsoft/vscode/blob/master/.yarnrc (choose correct tag) for target electron version
_electron=electron
pkgver=1.59.0
pkgrel=1
arch=('x86_64')
url='https://github.com/microsoft/vscode'
license=('MIT')
depends=($_electron 'libsecret' 'libx11' 'libxkbfile' 'ripgrep')
optdepends=('bash-completion: Bash completions'
            'zsh-completions: ZSH completitons'
            'x11-ssh-askpass: SSH authentication')
makedepends=('git' 'gulp' 'npm' 'python2' 'yarn' 'nodejs-lts-fermium')
conflicts=('code')
provides=('code' 'vscode')
install='code-transparent.install'
source=("$_pkgname::git+$url.git#tag=$pkgver"
        'code.js'
        'code.sh'
        'product_json.diff'
        'transparent.diff')
sha512sums=('SKIP'
            '6e8ee1df4dd982434a8295ca99e786a536457c86c34212546e548b115081798c5492a79f99cd5a3f1fa30fb71d29983aaabc2c79f4895d4a709d8354e9e2eade'
            'b8bdb0e53cf8748140ed444c9b02cb6a57a7e1e120d96861d4cc9f79744a987f0253c052a238c78aa2c3f86459c4afb6f3b687435f0588d8f640822a9908b257'
            'ae8a8a51c0aede5be2fa3579612c3eca622f001953870c7744f033938160fca4da416c0a9c1381910dca4907910dbf7cdebe50185c3c35931add79a5b83838d2'
            '04e92d392aba2bc8f8995d751d82b67bb5c10d6432b178fa1b51566872ee639a7f5dfbe9d62a2a9dc60b4f51a6d811cb99a15ca6d4e834375b1577f791ccd06a')

# Even though we don't officially support other archs, let's
# allow the user to use this PKGBUILD to compile the package
# for his architecture
case "$CARCH" in
  i686)
    _vscode_arch=ia32
    ;;
  x86_64)
    _vscode_arch=x64
    ;;
  armv7h)
    _vscode_arch=arm
    ;;
  *)
    # Needed for mksrcinfo
    _vscode_arch=DUMMY
    ;;
esac

prepare() {
  cd $_pkgname

  # Change electron binary name to the target electron
  sed -i "s|exec electron |exec $_electron |" ../code.sh
  sed -i "s|#!/usr/bin/electron|#!/usr/bin/$_electron|" ../code.js

  # This patch no longer contains proprietary modifications.
  # See https://github.com/Microsoft/vscode/issues/31168 for details.
  patch -p0 < ../product_json.diff

  # enable window transparency
  patch -p1 <../transparent.diff

  # Set the commit and build date
  local _commit=$(git rev-parse HEAD)
  local _datestamp=$(date -u -Is | sed 's/\+00:00/Z/')
  sed -e "s/@COMMIT@/$_commit/" -e "s/@DATE@/$_datestamp/" -i product.json

  # Build native modules for system electron
  local _target=$(</usr/lib/$_electron/version)
  sed -i "s/^target .*/target \"${_target//v/}\"/" .yarnrc

  # Patch appdata and desktop file
  sed -i 's|/usr/share/@@NAME@@/@@NAME@@|@@NAME@@|g
          s|@@NAME_SHORT@@|Code|g
          s|@@NAME_LONG@@|Code - OSS|g
          s|@@NAME@@|code-oss|g
          s|@@ICON@@|com.visualstudio.code.oss|g
          s|@@EXEC@@|/usr/bin/code-oss|g
          s|@@LICENSE@@|MIT|g
          s|@@URLPROTOCOL@@|vscode|g
          s|inode/directory;||' resources/linux/code{.appdata.xml,.desktop,-url-handler.desktop}

  sed -i 's|MimeType=.*|MimeType=x-scheme-handler/code-oss;|' resources/linux/code-url-handler.desktop

  # Add completitions for code-oss
  cp resources/completions/bash/code resources/completions/bash/code-oss
  cp resources/completions/zsh/_code resources/completions/zsh/_code-oss

  # Patch completitions with correct names
  sed -i 's|@@APPNAME@@|code|g' resources/completions/{bash/code,zsh/_code}
  sed -i 's|@@APPNAME@@|code-oss|g' resources/completions/{bash/code-oss,zsh/_code-oss}

  # Fix bin path
  sed -i "s|return path.join(path.dirname(execPath), 'bin', \`\${product.applicationName}\`);|return '/usr/bin/code';|g
          s|return path.join(appRoot, 'scripts', 'code-cli.sh');|return '/usr/bin/code';|g" \
          src/vs/platform/environment/node/environmentService.ts
}

build() {
  # https://github.com/mapbox/node-sqlite3/issues/1044
  mkdir -p path
  ln -sf /usr/bin/python2 path/python
  export PATH="$PWD/path:$PATH"

  cd $_pkgname

  yarn install --arch=$_vscode_arch

  # The default memory limit may be too low for current versions of node
  # to successfully build vscode. Change it if this number still doesn't
  # work for your system.
  mem_limit="--max_old_space_size=8192"

  if ! /usr/bin/node $mem_limit /usr/bin/gulp vscode-linux-$_vscode_arch-min
  then
      echo
      echo "*** NOTE: If the build failed due to running out of file handles (EMFILE),"
      echo "*** you will need to raise your max open file limit."
      echo "*** You can check this for more information on how to increase this limit:"
      echo "***    https://ro-che.info/articles/2017-03-26-increase-open-files-limit"
      exit 1
  fi
}

package() {
  # Install resource files
  install -dm 755 "$pkgdir"/usr/lib/$_pkgname
  cp -r --no-preserve=ownership --preserve=mode VSCode-linux-$_vscode_arch/resources/app/* "$pkgdir"/usr/lib/$_pkgname/

  # Replace statically included binary with system copy
  ln -sf /usr/bin/rg "$pkgdir"/usr/lib/code/node_modules.asar.unpacked/vscode-ripgrep/bin/rg

  # Install binary
  install -Dm 755 code.sh "$pkgdir"/usr/bin/code-oss
  install -Dm 755 code.js "$pkgdir"/usr/lib/$_pkgname/code.js
  ln -sf /usr/bin/code-oss "$pkgdir"/usr/bin/code

  # Install appdata and desktop file
  install -Dm 644 $_pkgname/resources/linux/code.appdata.xml "$pkgdir"/usr/share/metainfo/code-oss.appdata.xml
  install -Dm 644 $_pkgname/resources/linux/code.desktop "$pkgdir"/usr/share/applications/code-oss.desktop
  install -Dm 644 $_pkgname/resources/linux/code-url-handler.desktop "$pkgdir"/usr/share/applications/code-oss-url-handler.desktop
  install -Dm 644 VSCode-linux-$_vscode_arch/resources/app/resources/linux/code.png "$pkgdir"/usr/share/pixmaps/com.visualstudio.code.oss.png

  # Install bash and zsh completions
  install -Dm 644 $_pkgname/resources/completions/bash/code "$pkgdir"/usr/share/bash-completion/completions/code
  install -Dm 644 $_pkgname/resources/completions/bash/code-oss "$pkgdir"/usr/share/bash-completion/completions/code-oss
  install -Dm 644 $_pkgname/resources/completions/zsh/_code "$pkgdir"/usr/share/zsh/site-functions/_code
  install -Dm 644 $_pkgname/resources/completions/zsh/_code-oss "$pkgdir"/usr/share/zsh/site-functions/_code-oss

  # Install license files
  install -Dm 644 VSCode-linux-$_vscode_arch/resources/app/LICENSE.txt "$pkgdir"/usr/share/licenses/$_pkgname/LICENSE
  install -Dm 644 VSCode-linux-$_vscode_arch/resources/app/ThirdPartyNotices.txt "$pkgdir"/usr/share/licenses/$_pkgname/ThirdPartyNotices.txt
}


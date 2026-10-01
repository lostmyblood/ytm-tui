# Maintainer: Your Name <you@example.com>
pkgname=ytmtui
pkgver=1.0.0
pkgrel=1
pkgdesc='Minimal terminal UI for downloading YouTube Music (yt-dlp front end)'
arch=('any')
url='https://github.com/YOURUSER/ytmtui'
license=('MIT')
depends=('bash' 'yt-dlp' 'ffmpeg' 'jq')
optdepends=('chafa: cover art preview'
            'curl: cover art preview'
            'deno: JavaScript runtime for faster YouTube extraction'
            'python-mutagen: embed cover art in opus, flac and vorbis files')
source=("$pkgname-$pkgver.tar.gz::$url/archive/refs/tags/v$pkgver.tar.gz")
sha256sums=('SKIP')  # run `updpkgsums` before publishing

package() {
  cd "$pkgname-$pkgver"
  install -Dm755 ytm-dl.sh "$pkgdir/usr/bin/ytmtui"
  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}

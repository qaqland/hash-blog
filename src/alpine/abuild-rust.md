# Abuild & Rust(Cargo)

## rustup

## setup mirror

## keep offline

```bash
options="net" # cargo fetch
```

```bash
prepare() {
    default_prepare

    cargo fetch --target="$CTARGET" --locked
}
```

```bash
build() {
    cargo auditable build --frozen --release
    :
}
```

## clap(completions)

```bash
subpackages="
    $pkgname-bash-completion
    $pkgname-fish-completion
    $pkgname-zsh-completion
    "
```

```bash
build() {
    :
    ./target/release/mdbook completions bash > $pkgname.bash
    ./target/release/mdbook completions fish > $pkgname.fish
    ./target/release/mdbook completions zsh > $pkgname.zsh
}
```

```bash
package() {
    :
    install -Dm644 $pkgname.bash "$pkgdir"/usr/share/bash-completion/completions/$pkgname
    install -Dm644 $pkgname.fish "$pkgdir"/usr/share/fish/vendor_completions.d/$pkgname.fish
    install -Dm644 $pkgname.zsh "$pkgdir"/usr/share/zsh/site-functions/_$pkgname
}
```

## make patches

```bash
$ git init .
$ git add Cargo.toml Cargo.lock
$ git diff > ../../01-update-lock.patch
```

download patch from GitHub and GitLab

## tests

```bash
options="!check" # upstream has no test
```

```bash
check() {
    cargo test --frozen
}
```

```bash
$ cargo test -- --skip xxx
```

patches

## use system library

- env
- config.toml
- patch

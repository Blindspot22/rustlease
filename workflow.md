
```yml
name: Release Workflow

on:
  workflow_dispatch:
    inputs:
      type:
        description: 'Type of release (major/minor/patch)'
        required: true
        default: 'minor'
      dry_run:
        description: 'Dry run (true/false)'
        required: true
        default: true
      skip_tests:
        description: 'Skip tests (true/false)'
        required: true
        default: false

env:
  CARGO_TERM_COLOR: always

jobs:
  test:
    strategy:
      matrix:
        os: [ ubuntu-latest, macos-13, macos-14, windows-latest ]
    runs-on: ${{ matrix.os }}
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Ensure Cargo Directories Exist for Cache Restore
        run: |
          mkdir -p ~/.cargo/registry
          mkdir -p ~/.cargo/index
          mkdir -p target          
        shell: bash

      - name: Cache Cargo Registry, Index, Build
        uses: actions/cache@v3
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/index
            target            
          key: ${{ runner.os }}-${{ runner.arch }}-cargo-${{ hashFiles('**/Cargo.toml') }}
          restore-keys: |
            cargo-build-${{ runner.os }}-${{ runner.arch }}-${{ hashFiles('**/Cargo.toml') }}
            cargo-build-${{ runner.os }}-${{ runner.arch }}-            

      - name: Set up Rust
        uses: actions-rust-lang/setup-rust-toolchain@v1
        with:
          toolchain: stable

      - name: Build
        run: cargo build --verbose

      - name: Run Tests
        if: ${{ github.event.inputs.skip_tests == 'false' }}
        run: cargo test --verbose

  prepare-release:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      rr_cargo_version: ${{ steps.get-version.outputs.VERSION }}
      workflow_git_tag: ${{ steps.get-version.outputs.WORKFLOW_GIT_TAG }}
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
          token: ${{ secrets.RELEASE_TOKEN }}

      - name: Ensure Cargo Directories Exist for Cache Restore
        run: |
          mkdir -p ~/.cargo/registry
          mkdir -p ~/.cargo/index
          mkdir -p target          
        shell: bash

      - name: Cache Cargo Registry, Index, Build
        uses: actions/cache@v3
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/index
            target            
          key: ${{ runner.os }}-${{ runner.arch }}-cargo-${{ hashFiles('**/Cargo.toml') }}
          restore-keys: |
            cargo-build-${{ runner.os }}-${{ runner.arch }}-${{ hashFiles('**/Cargo.toml') }}
            cargo-build-${{ runner.os }}-${{ runner.arch }}-            

      - name: Cache Cargo Binaries
        uses: actions/cache@v3
        with:
          path: ~/.cargo/bin
          key: cargo-bin-${{ runner.os }}-${{ runner.arch }}-v1
          restore-keys: |
            cargo-bin-${{ runner.os }}-${{ runner.arch }}-            

      - name: Set up Rust
        uses: actions-rust-lang/setup-rust-toolchain@v1
        with:
          toolchain: stable

      - name: Install cargo-release
        run: |
          if ! command -v cargo-release &> /dev/null; then
            echo "Installing cargo-release..."
            cargo install cargo-release
          else
            INSTALLED_VERSION=$(cargo release --version || echo "unknown")
            echo "cargo-release already installed (version: $INSTALLED_VERSION). Skipping installation."
          fi          

      - name: Configure Git
        run: |
          git config --global user.name "GitHub Action"
          git config --global user.email "action@github.com"          

      - name: Update Cargo.toml version and push to GitHub
        run: |
          REL_TYPE=${{ github.event.inputs.type }}
          DRY_RUN=${{ github.event.inputs.dry_run }}

          # Execute version update
          if [ "$DRY_RUN" = "false" ]; then
            echo "Updating version in Cargo.toml"
            cargo release --verbose --execute --no-confirm $REL_TYPE --no-publish --no-verify
          else
            echo "Dry run: showing changes without executing"
            cargo release --verbose $REL_TYPE --no-publish --no-verify
          fi          

      - name: Get Version from Cargo.toml
        id: get-version
        run: |
          VERSION=$(grep '^version = ' Cargo.toml | sed -E 's/version = "(.*)"/\1/')
          echo "VERSION=$VERSION" >> "$GITHUB_OUTPUT"
          echo "WORKFLOW_GIT_TAG=v$VERSION" >> "$GITHUB_OUTPUT"          

  release:
    needs: prepare-release
    if: ${{ github.event.inputs.dry_run == 'false' }}
    strategy:
      matrix:
        os: [ ubuntu-latest, macos-13, macos-14, windows-latest ]
    runs-on: ${{ matrix.os }}
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Ensure Cargo Directories Exist for Cache Restore
        run: |
          mkdir -p ~/.cargo/registry
          mkdir -p ~/.cargo/index
          mkdir -p target          
        shell: bash

      - name: Cache Cargo Registry, Index, Build
        uses: actions/cache@v3
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/index
            target            
          key: ${{ runner.os }}-${{ runner.arch }}-cargo-${{ hashFiles('**/Cargo.toml') }}
          restore-keys: |
            cargo-build-${{ runner.os }}-${{ runner.arch }}-${{ hashFiles('**/Cargo.toml') }}
            cargo-build-${{ runner.os }}-${{ runner.arch }}-            

      - name: Set up Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable

      - name: Build Release
        run: cargo build --release --verbose

      - name: Copy release binary to root
        shell: bash
        run: |
          if [[ -f "target/release/example" ]]; then
            cp target/release/example .
          elif [[ -f "target/release/example.exe" ]]; then
            cp target/release/example.exe .
          else
            echo "No binary to copy for this OS."
          fi          

      - name: Get Build Info
        id: build-info
        shell: bash
        run: |
          OS=$(uname -s | tr '[:upper:]' '[:lower:]')
          ARCH=$(uname -m)
          TARGET=""
          case "$OS" in
            linux) TARGET="$ARCH-unknown-linux-gnu" ;;
            darwin) TARGET="$ARCH-apple-darwin" ;;
            msys*|cygwin*|mingw*) TARGET="$ARCH-pc-windows-msvc" ;;
          esac
          VERSION=$(grep '^version = ' Cargo.toml | sed -E 's/version = "(.*)"/\1/')
          FILENAME="rapidrecast-v${VERSION}-${TARGET}"
          echo "OS=$OS" >> "$GITHUB_ENV"
          echo "ARCH=$ARCH" >> "$GITHUB_ENV"
          echo "TARGET=$TARGET" >> "$GITHUB_ENV"
          echo "FILENAME=$FILENAME" >> "$GITHUB_ENV"          

      - name: Compress tar.gz
        uses: ksm2/archive-action@v1
        with:
          name: "${{ env.FILENAME }}"
          format: "tar.gz"
          include: "{example,example.exe,README.md,LICENSE}"

      - name: Compress zip
        uses: ksm2/archive-action@v1
        with:
          name: "${{ env.FILENAME }}"
          format: "zip"
          include: "{example,example.exe,README.md,LICENSE}"

      - name: Create or Update Release
        env:
          VERSION: ${{ needs.prepare-release.outputs.rr_cargo_version }}
          WORKFLOW_GIT_TAG: ${{ needs.prepare-release.outputs.workflow_git_tag}}
        uses: ncipollo/release-action@v1
        with:
          artifacts: "${{ env.FILENAME }}.tar.gz,${{ env.FILENAME }}.zip"
          allowUpdates: 'true'
          generateReleaseNotes: 'true'
          token: ${{ secrets.RELEASE_TOKEN }}
          tag: ${{ env.WORKFLOW_GIT_TAG }}
```
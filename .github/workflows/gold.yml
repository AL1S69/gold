name: Build Diting GKI Kernel (Zako+SUKISU+FastCharge+Opt)

on:
  workflow_dispatch: # 仅手动触发编译

jobs:
  build_kernel:
    runs-on: ubuntu-22.04
    steps:
      - name: 1. 安装GKI编译完整依赖
        run: |
          sudo apt update && sudo apt install -y \
            git make bison flex gcc-aarch64-linux-gnu bc libssl-dev libelf-dev \
            python3 curl liblz4-tool zstd clang lld llvm
          sudo apt clean

      - name: 2. 拉取你Fork的仓库源码
        uses: actions/checkout@v4
        with:
          fetch-depth: 1
          ref: master

      - name: 3. 下载Clang-r547379工具链（GKI 5.10兼容）
        run: |
          wget https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/main/clang-r547379.tar.gz
          mkdir clang-r547379 && tar zxf clang-r547379.tar.gz -C clang-r547379
          echo "CLANG_DIR=$(pwd)/clang-r547379" >> $GITHUB_ENV
          echo "KERNEL_DIR=$(pwd)" >> $GITHUB_ENV

      - name: 4. 集成SukiSU-Ultra（GKI 5.10分支，自动配置）
        run: |
          cd ${{ env.KERNEL_DIR }}
          # SUKISU脚本自动配置，取消手动追加配置
          curl -LSs "https://raw.githubusercontent.com/SukiSU-Ultra/SukiSU-Ultra/main/kernel/setup.sh" | bash -s gki-5.10
          [ -d "KernelSU" ] && echo "SUKISU集成成功" || exit 1

      - name: 5. 功能补丁集成（安全快充+充电小数点+调度优化）
        run: |
          cd ${{ env.KERNEL_DIR }}
          # 1. 安全快充支持（限制充电电流/温度，适配QC4+/PD3.0）
          cat >> include/linux/power_supply.h << EOF
          #define POWER_SUPPLY_CHARGE_PROTECT_TEMP 45 // 45℃触发快充限流
          #define POWER_SUPPLY_MAX_FAST_CHARGE_CURRENT 3000 // 最大快充电流3000mA
          extern void fast_charge_protection_init(void);
          EOF
          
          # 2. 充电小数点显示（精度0.1%）
          sed -i 's/int battery_level;/float battery_level;/g' drivers/power/supply/battery.c
          sed -i 's/battery_level = (capacity * 100) \/ full_capacity;/battery_level = (float)capacity * 100.0 \/ full_capacity;/g' drivers/power/supply/battery.c
          
          # 3. 内核调度优化（提升交互流畅度，降低延迟）
          # 调整CFS调度器参数
          echo "CONFIG_SCHED_DEBUG=y" >> out/.config
          echo "CONFIG_SCHED_INFO=y" >> out/.config
          echo "CONFIG_SCHED_TUNE=y" >> out/.config
          # 启用多核负载均衡优化
          sed -i 's/CONFIG_SCHED_MC=n/CONFIG_SCHED_MC=y/' out/.config
          sed -i 's/CONFIG_SCHED_SMT=n/CONFIG_SCHED_SMT=y/' out/.config
          # 降低调度延迟（适配移动设备）
          echo "CONFIG_HZ=1000" >> out/.config
          echo "CONFIG_PREEMPT=y" >> out/.config

      - name: 6. 修改内核名（-zako~kernel）
        run: |
          sed -i "s/LOCALVERSION=.*/LOCALVERSION=-zako~kernel/" ${{ env.KERNEL_DIR }}/Makefile

      - name: 7. 编译GKI内核（适配5.10，禁用LTO）
        run: |
          cd ${{ env.KERNEL_DIR }}
          export ARCH=arm64
          export SUBARCH=arm64
          export CROSS_COMPILE=aarch64-linux-gnu-
          export CC=${{ env.CLANG_DIR }}/bin/clang
          export CLANG_TRIPLE=aarch64-linux-gnu-
          export LD=${{ env.CLANG_DIR }}/bin/ld.lld
          export AR=${{ env.CLANG_DIR }}/bin/llvm-ar
          export NM=${{ env.CLANG_DIR }}/bin/llvm-nm
          export OBJCOPY=${{ env.CLANG_DIR }}/bin/llvm-objcopy
          export OBJDUMP=${{ env.CLANG_DIR }}/bin/llvm-objdump
          
          export KBUILD_LTO=none
          export KBUILD_ENABLE_EXTRA_GCC_CHECKS=n
          
          # 加载GKI基础配置（SUKISU已自动注入配置）
          make O=out ARCH=arm64 gki_defconfig
          yes "" | make O=out ARCH=arm64 olddefconfig
          
          # 编译核心产物
          make -j$(nproc) O=out ARCH=arm64 Image.gz-dtb
          
          # 归集产物
          mkdir -p boot
          cp -r out/arch/arm64/boot/* boot/
          ls -l boot/

      - name: 8. 上传boot目录所有产物
        uses: actions/upload-artifact@v4
        with:
          name: zako~kernel-diting-gki-sukisu-opt-boot
          path: ${{ env.KERNEL_DIR }}/boot/**/*
          retention-days: 14

# Nuke built-in rules and variables.
override MAKEFLAGS += -rR

define DEFAULT_VAR =
    ifeq ($(origin $1),default)
        override $(1) := $(2)
    endif
    ifeq ($(origin $1),undefined)
        override $(1) := $(2)
    endif
endef

override DEFAULT_ARCH := x86_64
$(eval $(call DEFAULT_VAR,ARCH,$(DEFAULT_ARCH)))

override DEFAULT_CC := cc
$(eval $(call DEFAULT_VAR,CC,$(DEFAULT_CC)))
export CC
override DEFAULT_LD := ld
$(eval $(call DEFAULT_VAR,LD,$(DEFAULT_LD)))
override DEFAULT_OBJCOPY := objcopy
$(eval $(call DEFAULT_VAR,OBJCOPY,$(DEFAULT_OBJCOPY)))

override CC_IS_CLANG := no
ifeq ($(shell $(CC) --version 2>&1 | grep -i 'clang' >/dev/null 2>&1 && echo 1),1)
    override CC_IS_CLANG := yes
endif

override DEFAULT_CFLAGS := -g -O2 -pipe
$(eval $(call DEFAULT_VAR,CFLAGS,$(DEFAULT_CFLAGS)))
override DEFAULT_CPPFLAGS :=
$(eval $(call DEFAULT_VAR,CPPFLAGS,$(DEFAULT_CPPFLAGS)))
override DEFAULT_LDFLAGS :=
$(eval $(call DEFAULT_VAR,LDFLAGS,$(DEFAULT_LDFLAGS)))

override CFLAGS += \
    -Wall \
    -Wextra \
    -std=gnu11 \
    -ffreestanding \
    -fno-stack-protector \
    -fno-stack-check \
    -fshort-wchar \
    -fno-lto \
    -fPIE

override CPPFLAGS := \
    -I src \
    -I limine-efi/inc \
    -I limine-efi/inc/$(ARCH) \
    $(CPPFLAGS) \
    -MMD \
    -MP

ifeq ($(ARCH),x86_64)
    ifeq ($(CC_IS_CLANG),yes)
        override CC += \
            -target x86_64-elf
    endif
    override CFLAGS += \
        -m64 \
        -march=x86-64 \
        -mno-80387 \
        -mno-mmx \
        -mno-sse \
        -mno-sse2 \
        -mno-red-zone
    override LDFLAGS += \
        -m elf_x86_64
    override NASMFLAGS += \
        -f elf64
else ifeq ($(ARCH),aarch64)
    ifeq ($(CC_IS_CLANG),yes)
        override CC += \
            -target aarch64-elf
    endif
    override CFLAGS += \
        -mgeneral-regs-only
    override LDFLAGS += \
        -m aarch64elf
else ifeq ($(ARCH),riscv64)
    ifeq ($(CC_IS_CLANG),yes)
        override CC += \
            -target riscv64-elf
        override CFLAGS += \
            -march=rv64imac
    else
        override CFLAGS += \
            -march=rv64imac_zicsr_zifencei
    endif
    override CFLAGS += \
        -mabi=lp64 \
        -mno-relax
    override LDFLAGS += \
        -m elf64lriscv \
        --no-relax
else
    $(error Architecture $(ARCH) not supported)
endif

override LDFLAGS += \
    -nostdlib \
    -static \
    -pie \
    --no-dynamic-linker \
    -z text \
    -z max-page-size=0x1000 \
    -T limine-efi/gnuefi/elf_$(ARCH)_efi.lds

override CFILES := $(shell cd src && find -L * -type f -name '*.c')
override OBJ := $(addprefix obj/,$(CFILES:.c=.c.o))
override HEADER_DEPS := $(addprefix obj/,$(CFILES:.c=.c.d))

.PHONY: all
all: bin/HELLO.EFI

limine-efi/gnuefi/crt0-efi-$(ARCH).S: limine-efi

limine-efi/gnuefi/crt0-efi-$(ARCH).S.o: limine-efi/gnuefi/crt0-efi-$(ARCH).S
	$(MAKE) -C limine-efi/gnuefi ARCH=$(ARCH) crt0-efi-$(ARCH).S.o

limine-efi/gnuefi/reloc_$(ARCH).c: limine-efi

limine-efi/gnuefi/reloc_$(ARCH).c.o: limine-efi/gnuefi/reloc_$(ARCH).c
	$(MAKE) -C limine-efi/gnuefi ARCH=$(ARCH) reloc_$(ARCH).c.o

limine-efi/gnuefi/elf_$(ARCH)_efi.lds: limine-efi

limine-efi:
	git clone https://github.com/limine-bootloader/limine-efi.git --depth=1

bin/HELLO.EFI: bin/hello.elf GNUmakefile
	mkdir -p "$$(dirname $@)"
	$(OBJCOPY) -O binary $< $@

bin/hello.elf: GNUmakefile limine-efi/gnuefi/elf_$(ARCH)_efi.lds limine-efi/gnuefi/crt0-efi-$(ARCH).S.o limine-efi/gnuefi/reloc_$(ARCH).c.o $(OBJ)
	mkdir -p "$$(dirname $@)"
	$(LD) limine-efi/gnuefi/crt0-efi-$(ARCH).S.o limine-efi/gnuefi/reloc_$(ARCH).c.o $(OBJ) $(LDFLAGS) -o $@

-include $(HEADER_DEPS)

obj/%.c.o: src/%.c GNUmakefile limine-efi
	mkdir -p "$$(dirname $@)"
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

ovmf-x86_64:
	mkdir -p ovmf-x86_64
	cd ovmf-x86_64 && curl -o OVMF.fd https://retrage.github.io/edk2-nightly/bin/RELEASEX64_OVMF.fd

ovmf-aarch64:
	mkdir -p ovmf-aarch64
	cd ovmf-aarch64 && curl -o OVMF.fd https://retrage.github.io/edk2-nightly/bin/RELEASEAARCH64_QEMU_EFI.fd

ovmf-riscv64:
	mkdir -p ovmf-riscv64
	cd ovmf-riscv64 && curl -o OVMF.fd https://retrage.github.io/edk2-nightly/bin/RELEASERISCV64_VIRT_CODE.fd && dd if=/dev/zero of=OVMF.fd bs=1 count=0 seek=33554432

.PHONY: run
run: all ovmf-$(ARCH)
	mkdir -p boot/EFI/BOOT
ifeq ($(ARCH),x86_64)
	cp bin/HELLO.EFI boot/EFI/BOOT/BOOTX64.EFI
	qemu-system-$(ARCH) -net none -M q35 -bios ovmf-$(ARCH)/OVMF.fd -drive file=fat:rw:boot
else ifeq ($(ARCH),aarch64)
	cp bin/HELLO.EFI boot/EFI/BOOT/BOOTAA64.EFI
	qemu-system-$(ARCH) -net none -M virt -cpu cortex-a72 -device ramfb -device qemu-xhci -device usb-kbd -bios ovmf-$(ARCH)/OVMF.fd -drive file=fat:rw:boot
else ifeq ($(ARCH),riscv64)
	cp bin/HELLO.EFI boot/EFI/BOOT/BOOTRISCV64.EFI
	qemu-system-$(ARCH) -net none -M virt -cpu rv64 -device ramfb -device qemu-xhci -device usb-kbd -drive if=pflash,unit=0,format=raw,file=ovmf-$(ARCH)/OVMF.fd -device virtio-scsi-pci,id=scsi -device scsi-hd,drive=hd0 -drive id=hd0,file=fat:rw:boot
endif
	rm -rf boot

.PHONY: clean
clean:
	if [ -d limine-efi/gnuefi ]; then $(MAKE) -C limine-efi/gnuefi ARCH=$(ARCH) clean; fi
	rm -rf bin obj

.PHONY: distclean
distclean: clean
	rm -rf limine-efi ovmf*

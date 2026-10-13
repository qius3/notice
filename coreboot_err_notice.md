```bash
make menuconfig
src/mainboard/agilerobots/Kconfig:17:warning: no new line at end of file
src/mainboard/agilerobots/Kconfig:17: 'if' in different file than 'if'
src/mainboard/agilerobots/Kconfig:3: location of the 'if'
src/Kconfig:712: 'menu' in different file than 'menu'
src/mainboard/agilerobots/Kconfig:3: location of the 'menu'
make: *** [build/util/kconfig/Makefile.real:55: menuconfig] Error 1
```

* no new line at end of file
Kconfig 解析器把换行符当作一个重要的“语句结束符”。


* power save 模式

	```

	关闭power save模式，关闭低功耗在工程选项里predefined把POWER_SAVING去掉就可以
	Properties->Build->Advanced Options->Predefined Symbols,
	添加POWER_SAVING模式。
	
	```

* sdk移植到CC2640
	
	```

	sdk代码烧写到cc2640上不起作用。
	需要添加相关代码和宏,涉及代码比较多，需要TI支持。
	宏定义需要在Predefined Symbols添加CC2650DK_5IS。

	```

* 编译出错，提示找不到一些依赖库。


	```

	Properties->General->Project->Connection选择合适的配置。一般用default。选错就会这样。

	```

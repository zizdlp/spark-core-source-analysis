# Spark存储体系

块管理器 `BlockManager` 是 Spark存储体系中的核⼼组件，因此本章内容主要围绕`BlockManager`展开。DriverApplication和Executor都会创建BlockManager。

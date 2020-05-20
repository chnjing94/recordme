### 用 java -jar启动jar包形式的SpringBoot应用，具体执行流程

java解压后有三个子文件夹BOOT-INF/，META-INF/，org/。根据JAR规范，会首先执行META-INF/MANIFEST.MF里指定的Main-Class，SpringBoot指定的Main-Class是/org/springframework/boot/loader/JarLauncher，JarLauncher.class里面指定了ClassPath，以及lib文件目录，在它的main方法里执行launch方法来启动应用程序。具体执行的入口方法是在MANIFEST.MF里指定的Start-Class，也就是我们用@SpringBootApplication标注的类的main函数(如果该类里没有main函数，就会扫描项目所有类中的main函数，如果有多个main函数就会报错)。

### SpringBoot怎么读取注解

通过AnnotationMetadata API获取，该API有两种实现方式，一种是通过Java反射实现，一种通过ASM实现。

ASM的性能会比反射方式好（反射需要排除Java标准注解）。反射要求类被ClassLoader加载，会导致package下所有类被加载，而ASM是按需加载。


# Java日志框架演进历程及特点：从JCL到SLF4J，Logback、Log4J与Log4J2的诞生以及JCL-Over-SLF4J的作用

## 背景
在Java开发领域中，为了满足不同场景下的日志需求，各类日志框架应运而生，其中不乏Jakarta Commons Logging (JCL)、Simple Logging Facade for Java (SLF4J)、Logback、Apache Log4J、Log4J2和JCL-Over-SLF4J等重要角色。本文将详述这些框架的诞生背景、各自特点及其应用场景。
## 1. Jakarta Commons Logging (JCL)
JCL是早期Java社区为了解决各日志实现之间的互操作性问题而推出的轻量级日志接口库，由Apache Jakarta Commons项目提供。JCL为其他日志框架提供了一个统一的接口，然后根据实际环境选择不同的日志实现（如Log4J）。然而，由于JCL依赖类加载器顺序来决定使用哪种日志实现，这在多模块、复杂环境中容易导致问题，因此催生了更优解决方案的需求。
## 2. Simple Logging Facade for Java (SLF4J)
SLF4J正是在这种背景下诞生的，作为对JCL等早期日志框架局限性的改进方案。它提供了更为简洁、灵活且无侵入性的日志门面层，允许开发者在部署时动态绑定到任意支持的日志实现。SLF4J的设计目标是降低日志API的耦合度，并提升可扩展性和兼容性，使其成为现代Java项目的理想选择。
## 3. Logback
Logback是由Ceki Gülcü（同时也是SLF4J的作者）创建的新一代日志框架，旨在解决Log4J的部分性能瓶颈和设计局限。Logback不仅实现了SLF4J API，还具有高度优化的性能、丰富的配置选项和强大的扩展能力。许多团队选择Logback作为其默认的日志实现工具。
## 4. Apache Log4J 与 Log4J2
Apache Log4J曾一度是Java世界中最流行的日志框架之一，以其灵活性和模块化设计著称。然而，随着时间推移，原版Log4J暴露出了一些性能和功能上的短板。于是，Apache基金会推出了Log4J2，这是一个经过彻底重构并优化的新版本，引入了许多新特性，如异步日志处理、垃圾回收友好设计、众多插件支持等，极大地提升了性能和稳定性。
## 5. JCL-Over-SLF4J
鉴于很多遗留系统或第三方库仍依赖于JCL进行日志记录，为帮助这类项目顺利过渡至SLF4J体系，JCL-Over-SLF4J作为适配器被创造出来。当项目中引入JCL-Over-SLF4J时，所有针对JCL的调用会自动重定向至SLF4J，使得开发者可以继续利用现有代码结构，同时享受到SLF4J带来的便利和优势。
## 总结
在研发过程中，推荐采用SLF4J作为日志门面层，以便灵活地切换底层日志实现。
根据项目性能需求和团队熟悉程度，可以选择Logback或Log4J2作为具体的日志实现工具。
对于那些依赖于JCL的遗留库或第三方组件，通过引入JCL-Over-SLF4J实现无缝迁移至SLF4J体系。
常见日志解决方案：SLF4J+Log4J、SLF4J+Logback、JCL-Over-SLF4J+SLF4J+Log4J、JCL-Over-SLF4J+SLF4J+Logback等。
## 相关问题
### JCL为什么不选择Maven来定义日志门面和日志实现
时间因素：JCL是在Maven流行之前就已经存在的项目。在JCL被设计和开发的时候，Maven可能还没有被广大开发者接受或成为主流的项目构建和依赖管理工具。因此，JCL的开发者可能基于当时的技术环境和工具链，选择了其他方式来定义和管理项目的依赖和构建。
依赖管理的独立性：JCL作为一个通用的日志门面，旨在提供一个统一的接口，使得开发者可以无缝切换不同的日志实现。如果JCL使用Maven来定义日志门面和日志实现，这可能会限制其在不同项目、不同构建系统或不同环境中的使用。选择不依赖于特定构建工具的方式定义，使得JCL具有更大的灵活性和独立性。
社区和维护：Apache作为一个开源社区，有着自己的开发流程和规范。在选择项目构建和依赖管理工具时，可能还需要考虑社区的习惯、经验以及维护成本等因素。JCL作为Apache的一部分，其开发和维护依赖于社区中的贡献者，因此可能选择了一个更符合社区需求和习惯的方式来定义日志门面和实现。
### JCL-Over-SLF4J 使用的注意事项
JCL-Over-SLF4J 可以与 Log4J、Log4J2 或 Logback 同时存在于一个项目中，但为了确保日志系统的正常运行和避免冲突，需要遵循特定的配置步骤和注意事项。
JCL-Over-SLF4J 是一个适配器，它的作用是将原本依赖于 Jakarta Commons Logging (JCL) API 的代码重定向至 SLF4J。当存在第三方库使用 JCL 进行日志记录时，引入 JCL-Over-SLF4J 可以让这些日志请求通过 SLF4J 进行处理。
同时，如果项目选择 Log4J、Log4J2 或 Logback 作为最终的日志实现，需要确保：
在项目的类路径（Classpath）上首先加载 JCL-Over-SLF4J，这样对 JCL 的调用会被拦截并转发到 SLF4J。
确保仅有一个 SLF4J 实现绑定在类路径上。例如，如果选择了 Logback，则需包含 slf4j-api 和 logback-classic 库；若选择 Log4J 或 Log4J2，则分别需要对应的 slf4j-log4j12 或 slf4j-log4j13（针对 Log4J），以及 slf4j-api 和 log4j-slf4j-impl（针对 Log4J2）。
注意避免类加载器顺序导致的问题，确保适配器和实际日志实现被正确加载。
总的来说，JCL-Over-SLF4J 可以与其它日志实现共存，但必须合理配置类路径和相关依赖，以确保整个日志系统能够按照预期进行工作。
### Apache Commons Logging 和 Jakarta Commons Logging 的区别
Apache Commons Logging 和 Jakarta Commons Logging 实际上指的是同一个项目。起初，这个项目是作为 Apache Jakarta 项目的子项目而存在的，并被命名为 Jakarta Commons Logging（JCL），旨在为 Java 应用程序提供一个统一的日志接口层，以解耦具体日志实现框架的依赖。
随着 Apache 软件基金会内部结构的变化，Jakarta 项目的部分组件被重组并迁移到了 Apache Commons 子项目下。因此，原名为 Jakarta Commons Logging 的项目在迁移后就成为了 Apache Commons Logging，但人们仍然习惯性地使用 JCL 这个简称来指代它。
## 相关文档
jcl-over-slf4j log桥接工具简介：[https://blog.csdn.net/zgmzyr/article/details/10464555](https://blog.csdn.net/zgmzyr/article/details/10464555)
log日志框架和LocationAwareLogger问题：[https://www.cnblogs.com/widget90/p/7610034.html](https://www.cnblogs.com/widget90/p/7610034.html)
SLF4J 的几种实际应用模式--之三：JCL-Over-SLF4J+SLF4J：[https://yanbin.blog/jcl-over-slf4j-slf4j/](https://yanbin.blog/jcl-over-slf4j-slf4j/)

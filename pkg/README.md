pkg/ is a collection of utility packages used by the InfluxDB project without being specific to its internals.

Utility packages are kept separate from the InfluxDB core codebase to keep it as small and concise as possible.  If some utilities grow larger and their APIs stabilize, they may be moved to their own repository under the InfluxDB organization, to facilitate re-use by other projects. However that is not the priority.

Because utility packages are small and neatly separated from the rest of the codebase, they are a good place to start for aspiring maintainers and contributors. Get in touch if you want to help maintain them!

pkg /是InfluxDB项目使用的实用程序包的集合，而不是特定于其内部。

实用程序包与InfluxDB核心代码库分开保存，以使其尽可能小巧简洁。 如果某些实用程序变得更大并且其API稳定，则可以将它们移动到InfluxDB组织下的自己的存储库中，以便于其他项目的重用。 然而，这不是优先事项。

因为实用程序包很小并且与代码库的其余部分完全分离，所以它们是有抱负的维护者和贡献者的好地方。 如果您想帮助维护它们，请联系我们！
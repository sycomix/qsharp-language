# Namespaces

At the top-level, a Q# program consists of a set of namespaces; namespaces are the only top-level elements, and anything else needs to be contained in a namespace. 
Q# does not support nested namespaces.

Namespaces can span multiple files. By default, everything declared within the same namespace can be accessed without further qualification, whereas declarations in a different namespace can only be used either by qualifying their name with the name of the namespace they belong to, or by opening that namespace before use, as it is done for `Microsoft.Quantum.Preparation` in [example program](https://github.com/microsoft/qsharp-language/blob/main/Specifications/Language/ProgramStructure/README.md#program-execution). Such `open` directives need to precede any other namespace elements, and are valid throughout the namespace piece in that file only. 

It is possible to define an alternative, usually shorter, name for a particular namespace to avoid having to type out the full name but still distinguish where certain elements came from. This is done, e.g., for `Microsoft.Quantum.Diagnostics` in the [example program](https://github.com/microsoft/qsharp-language/blob/main/Specifications/Language/ProgramStructure/README.md#program-execution). Defining namespace aliases is particularly helpful in combination with the code completion functionality provided by the Q# extensions available for Visual Studio Code and Visual Studio; If the extension is installed, typing the namespace alias followed by a dot will show a list of all available elements in that namespace that can be used at the current location.  

Aside from `open` directives, namespaces can also contain operation, function and type declarations. These may occur in any order, and are recursive by default, meaning they can be used in any order and call itself. 
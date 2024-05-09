# Developer Portal

This project stores the files for the [Espressif Developer Portal][] website. GitHub Actions statically generate the website using Hugo and pushes it to a web server for online hosting.

[Espressif Developer Portal]: https://developer.espressif.com/

If you want to contribute to this project, it would be nice to see the rendered version. The website can be easily rendered on your local machine using the following steps:

1. Clone this repository using `git clone --recursive`.
2. Install Hugo following the [instructions](https://gohugo.io/installation/).
3. In the project folder, run `hugo server` and open the provided local web address, usually http://localhost:1313/.

See also the Contribution Guide articles:

- [Content contribution workflow](./content/pages/contribution_guide/content-contrib-workflow/index.md)
- [Content writing workflow](./content/pages/contribution_guide/content-writing-workflow/index.md)

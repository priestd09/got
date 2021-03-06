# Go T(emplates)

The native Go [`html/template` library](https://golang.org/pkg/html/template/) is powerful. Template inheritance, helper functions, and proper sanitizing of HTML, CSS, and Javascript are all provided. However, using `html/template` requires a bit of boilerplate (and forethought) which this package provides in a clever, minimal wrapper.

1) This library is for people who want use `html/template`.
2) This package requires you to organize your templates _on-disk_ in a certain, logical way.


## Usage

Specify the directory containing your templates, and the file extension for template files.

    templates, err := got.New("templates/", ".html")

Once loaded, rendering templates inside a handler is as easy as passing the `http.ResponseWriter`, template name, data, and a status.

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
      data = struct{ Name string }{"Bob"}
      err := templates.Render(w, "home", data, http.StatusOK)

      if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
      }
    })

Should an error be encountered, `got` will prevent a partial response from being sent, allowing you to handle the error (as shown above).


## Conventions

`xeoncross/got` expects layout files to live in one of three folders based on their usage:

- `pages` are the "main content" for your routes
- `includes` are shared code blocks such as "sidebars" or "ads"
- `layouts` are the different page "shells" used by pages.

Each page is the "starting" point. For example, you might have a file structure like this:

    pages/
      home.html
      about.html
      profile.html
      posts.html
    layouts/
      main.html
      forum.html
    includes/
      sidebar.html

In this example, imagine the "profile" and "posts" page use the `forum.html` layout, while the "home" & "about" pages use `main.html` + `sidebar.html`. Every page has access to all includes.


## The Pain Point: Inheritance

With plain `html/template` you can't specify a template parent from the child. Instead, you have to load templates _backwards_ by loading the child, then having the parent template.Execute() to render the child correctly inside it.

    t, _ := template.ParseFiles("base.tmpl", "about.tmpl")
    t.Execute(w, nil)

- https://blog.questionable.services/article/approximating-html-template-inheritance/
- https://www.kylehq.com/2017/05/golang-templates---what-i-missed/ ([gist](https://gitlab.com/snippets/1662623))


## Solution

We solve this problem by adding a simple [template comment](https://golang.org/pkg/text/template/#hdr-Actions) to the child:

    {/* use mobilelayout */}

This comment is removed by `html/template` in the final output, but tells `got` to load this child template inside `mobilelayout.html`.

## Subfolders

Simple sites can easily fit everything under the `pages`, `includes`, and `layouts` namespaces. However, for better organization of your HTML snippets you can also use subfolders. For example, you might have custom sidebar modules you wish to isolate to their own files. Including items in subfolders (and beyond) is as easy as regular files. Imagine you had the following file:

    /templates/includes/sidebar/active_users.html

You can access this in your templates using the string `includes/sidebar/active_users`.

## Benchmarks

This library adds almost no overhead to `html/template` for rendering templates. This package is all about *layout conventions* without interfering with performance.

    $ go test -bench=. --benchmem -test.cpu=1

    BenchmarkCompile         	  300000	      4079 ns/op	    1256 B/op	      30 allocs/op
    BenchmarkNativeTemplates 	  300000	      4028 ns/op	    1256 B/op	      30 allocs/op

This library is as fast as `html/template` because the organizational sorting and inheritance calculations are performed on the initial load.


## Template Functions (Recommended)

Template functions provide handy helpers for doing common tasks. The [Masterminds/sprig](https://github.com/Masterminds/sprig) package contains +100 helper functions (inspired by `underscore.js`) you can use to augment your templates.

If building HTML forms, or using the CSS framework Bootstrap, you might want to look at [gobuffalo/tags](https://github.com/gobuffalo/tags) for helper functions to generate HTML.


## Alternatives

There are a [number of template engines](https://awesome-go.com/#template-engines) available should you find `html/template` lacking for your use case. @SlinSo has put together a good [benchmark of these different template engines](https://github.com/SlinSo/goTemplateBenchmark).

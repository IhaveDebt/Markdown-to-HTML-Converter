package main

import (
	"flag"
	"fmt"
	"html/template"
	"io"
	"io/ioutil"
	"os"
	"path/filepath"
	"strings"

	blackfriday "github.com/russross/blackfriday/v2"
)

var tmpl = `<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>{{ .Title }}</title>
<style>
body{max-width:900px;margin:2rem auto;font-family:Inter,system-ui,Arial,sans-serif;line-height:1.6;padding:0 1rem}
pre{background:#f5f5f7;padding:1rem;border-radius:8px;overflow:auto}
code{font-family: SFMono-Regular,Menlo,monospace}
</style>
</head>
<body>
<article>{{ .Body }}</article>
</body>
</html>
`

type Page struct {
	Title string
	Body  template.HTML
}

func convertMarkdown(md []byte) []byte {
	html := blackfriday.Run(md, blackfriday.WithExtensions(
		blackfriday.CommonExtensions|blackfriday.AutoHeadingIDs|blackfriday.FencedCode,
	))
	return html
}

func processFile(path, srcDir, outDir string, wrapPage bool) error {
	b, err := ioutil.ReadFile(path)
	if err != nil {
		return err
	}
	html := convertMarkdown(b)

	rel, _ := filepath.Rel(srcDir, path)
	outPath := filepath.Join(outDir, strings.TrimSuffix(rel, filepath.Ext(rel))+".html")
	if err := os.MkdirAll(filepath.Dir(outPath), 0o755); err != nil {
		return err
	}

	if wrapPage {
		t := template.Must(template.New("page").Parse(tmpl))
		f, err := os.Create(outPath)
		if err != nil {
			return err
		}
		defer f.Close()
		data := Page{Title: strings.TrimSuffix(filepath.Base(rel), filepath.Ext(rel)), Body: template.HTML(html)}
		return t.Execute(f, data)
	}

	return ioutil.WriteFile(outPath, html, 0o644)
}

func main() {
	src := flag.String("src", "content", "source directory with .md files")
	out := flag.String("out", "public", "output directory for HTML files")
	page := flag.Bool("page", true, "wrap HTML in full HTML page boilerplate")
	flag.Parse()

	// fallback single-file mode: if src is a file, convert single file to stdout or out path
	info, err := os.Stat(*src)
	if err != nil {
		fmt.Fprintln(os.Stderr, "Source not found:", err)
		os.Exit(1)
	}

	if info.Mode().IsRegular() {
		md, err := ioutil.ReadFile(*src)
		if err != nil {
			fmt.Fprintln(os.Stderr, "read error:", err)
			os.Exit(1)
		}
		outHTML := convertMarkdown(md)
		if *page {
			t := template.Must(template.New("page").Parse(tmpl))
			data := Page{Title: strings.TrimSuffix(filepath.Base(*src), filepath.Ext(*src)), Body: template.HTML(outHTML)}
			if *out == "-" {
				t.Execute(os.Stdout, data)
			} else {
				if err := os.MkdirAll(filepath.Dir(*out), 0o755); err != nil {
					fmt.Fprintln(os.Stderr, "mkdir error:", err); os.Exit(1)
				}
				f, _ := os.Create(*out)
				defer f.Close()
				t.Execute(f, data)
			}
		} else {
			if *out == "-" {
				os.Stdout.Write(outHTML)
			} else {
				if err := ioutil.WriteFile(*out, outHTML, 0o644); err != nil {
					fmt.Fprintln(os.Stderr, "write error:", err); os.Exit(1)
				}
			}
		}
		fmt.Println("Converted single file.")
		return
	}

	// directory mode
	err = filepath.Walk(*src, func(path string, info os.FileInfo, err error) error {
		if err != nil { return err }
		if info.IsDir() { return nil }
		ext := strings.ToLower(filepath.Ext(path))
		if ext == ".md" || ext == ".markdown" {
			if err := processFile(path, *src, *out, *page); err != nil {
				fmt.Fprintln(os.Stderr, "failed:", path, err)
			} else {
				fmt.Println("Generated:", path)
			}
		}
		return nil
	})
	if err != nil {
		fmt.Fprintln(os.Stderr, "walk error:", err)
		os.Exit(1)
	}
	fmt.Println("Done – output in", *out)
}

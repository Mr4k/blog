Dev Build: `pelican content -s pelicanconf.py -t theme/`  
Production Build: `pelican content -s publishconf.py -t theme/`  
Run Local Dev Server: `cd output && python -m http.server`  
Publish to gh-pages branch:  
`pelican content -o output -s pelicanconf.py`  
`ghp-import output -b gh-pages`   
`git push origin gh-pages`  
Alternatively just use the commands in `task.py`

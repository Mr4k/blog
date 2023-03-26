Dev Build: `pelican -r content -s pelicanconf.py -t theme/`  
Production Build: `pelican content -s publishconf.py -t theme/`  
Run Local Dev Server: `cd output && python -m http.server`  
Publish to gh-pages branch:  
`pelican content -s publishconf.py -t theme/`  
`ghp-import output -b gh-pages`   
`git push origin gh-pages`  
Move CNAME from main to gh-pages as it will have been deleted. (TODO have this happen automatically)
Alternatively just use the commands in `task.py`

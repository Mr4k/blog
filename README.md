Setup uv environment:
- If there is no venv `uv venv`
- `source .venv/bin/activate`
- `uv sync`
- `uv pip install "pelican[markdown]"`
- `uv pip install "pelican-render-math"`
- `uv pip install "pelican-simple-footnotes"`

Dev Build: `pelican -r content -s pelicanconf.py -t theme/`  
Production Build: `pelican content -s publishconf.py -t theme/`  
Run Local Dev Server: `cd output && python -m http.server`  
Publish to gh-pages branch:  
`pelican content -s publishconf.py -t theme/`  
`ghp-import output -b gh-pages` 
`git checkout gh-pages`
`echo -e "www.peterstefek.me\n" > CNAME`
`git add CNAME`
`git commit -m 'readded CNAME'`
`git push origin gh-pages`  

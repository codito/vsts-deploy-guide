We use [mkdocs](http://www.mkdocs.org/) to convert markdown to this beautiful
guide. It's really simple to start.

```
# Let's assume you've already installed python on your box
# Python and pip are in your PATH environment variable
git clone http://github.com/codito/vso-deploy-guide.git vso-deploy-guide

cd vso-deploy-guide

pip install -r requirements.txt

mkdocs serve
```

Now browse to [http://127.0.0.1:8000](http://127.0.0.1:8000). You'll see a local version of this guide.
Open your favorite editor and write away. Any changes will automatically reflect
on the local server.

Once you're happy about the changes, create a pull request in github. Let's
spread the love together!

(installing-code)=
# Installing code

If you want to run the code locally, first get a Python distribution. I use [Miniconda](https://docs.anaconda.com/miniconda/). Alternatively, you could run it on Google Colab.

Install required packages. Using Python built-in tools you could do it like this:

```
pip install jinja2 notebook ipykernel matplotlib numpy wordcloud
```

Using conda, you could create an environment like this:

```
conda create -n snufa python=3 jinja2 notebook ipykernel matplotlib numpy wordcloud
conda activate snufa
```

Then run the notebook you're interested in, either using Jupyter command line tool:

```
jupyter notebook
```

Or you could use ``vscode`` and set it to use this kernel.
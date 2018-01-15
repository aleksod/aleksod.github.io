---
layout: post
title: Dockerizing Your Python App (oh, and Mining Text from PDFs Along the Way)
---

![title image](http://mikebirdgeneau.com/theme/images/docker_python.jpg)

# Intro
I have been out of Metis for several months now, but I haven't been standing still. Apart from a constant applying to jobs and attending Python and Data Science related meetups, I also have been doing some volunteering and freelancing on the side (more on that in later posts). Today I would like to tell you about something I encountered during one of my freelancing jobs.

# The Problem
My client had hundreds of PDF files from which he needed to extract some textual information. Naturally, my first Pythonic reflex was to look for a suitable module that would get the job done. I surveyed a couple of such modules, namely [`textract`](http://textract.readthedocs.io/en/latest/), [`PDFPlumber`](https://github.com/jsvine/pdfplumber), and [`PyPDF2`](https://github.com/mstamy2/PyPDF2). The first two gave me trouble, so those were skipped in favor of `PyPDF2` because I thought this would be troublesome when the time comes to deploy my code to the client's machine:
- `textract` gave me the following error when I tried to `pip install` it:
```
1 error generated.
    error: command 'gcc' failed with exit status 1
```
I think this may have something to do with code compilation on my machine. Not assuming anything about technical competencies of my client, imagine how hard this would be to make the code run on his machine.
- `PDFPlumber` installed nicely with no error messages. However, when I tried reading a PDF file with
```python
with pdfplumber.open("file.pdf") as pdf:
    first_page = pdf.pages[0]
    print(first_page.chars[0])
```
it yelled at me:
```
PDFSyntaxError: Catalog not found!
```
so `PDFPlumber` was skipped as well.

# PyPDF2 in Action
`PyPDF2`, in contrast, was installed with no complaints and I was able to read PDFs via
```python
pdf = PyPDF2.PdfFileReader("file.pdf")
```

Since I had many files to read, I made the script to walk through all PDF files in a directory by using `glob` module:
```python
import PyPDF2
import glob

path = 'data/'
result = list()

for filename in glob.glob(path + "*.pdf"):
    try:
        pdf = PyPDF2.PdfFileReader(filename)
```

To extract text from a PDF file using `PyPDF2` module, I had to first indicate which page I am interested in. Luckily, my client's PDF files were all pretty standardized and all had one page only. Therefore, using the following code helped me to extract the text and to get rid of all line break characters along the way, just to clean it up a little bit:
```python
text = pdf.getPage(0).extractText()
text = " ".join(text.split())
```

The client wanted an Excel table populated with certain information from the files, so I used `re` and `pandas` modules for regular expressions' text extraction and dataframe manipulations, respectively:
```python
import PyPDF2
import re
import pandas as pd
import glob

path = 'data/'
result = list()

for filename in glob.glob(path + "*.pdf"):
    try:

        pdf = PyPDF2.PdfFileReader(filename)
        text = pdf.getPage(0).extractText()
        text = " ".join(text.split())

        try:
            field_to_populate_1 = re.findall(r"some RegEx expression", text)[0]
        except IndexError:
            field_to_populate_1 = "NOT FOUND"

        try:
            field_to_populate_2 = re.findall(r"another RegEx expression", text)[0]
        except IndexError:
            field_to_populate_2 = "NOT FOUND"
        
        result.append({
                "Field 1" : field_to_populate_1,
                "Field 2" : field_to_populate_2
                })
        
    except PyPDF2.utils.PdfReadError:
        print(filename, "could not be read.")

df = pd.DataFrame(result)

columns = [
    "Field 1",
    "Field 2"
]

df = df[columns]

writer = pd.ExcelWriter('data/output.xlsx')
df.to_excel(writer)
writer.save()
```
As you can see, I used a heavy dose of Python exception handling via `try:` and `except:` to make sure everything went as smoothly as it could and no fields would go unpopulated. The resultant `output.xlsx` file would be deposited into the same folder where PDF files were stored.

# Deploying my Python Script on the Client's Machine
Now came the tricky part. How do I make sure everything runs smoothly on the other end and the script works exactly as it did on my machine?  

In order for the transition to go smoothly, I stayed in a chatroom with my client and walked him through the execution process. At first I thought hosting my code privately on BitBucket and inviting him to be able to clone it would be a part of the solution. Unfortunately, the client did not have GIT set up, so he had to manually download the script every time I made changes to it. And changes did have to be made, as we soon discovered few quirks due to his files not always being structured the same way. I.e. what PyPDF2 saw did not always have the same RegEx pattern, so I had to broaden my patterns' scope. This resulted in many back and forths with the client. On top of that, for some reason his `pandas` module also needed to have `openpyxl` in order to work, something that was not the case on my own machine.
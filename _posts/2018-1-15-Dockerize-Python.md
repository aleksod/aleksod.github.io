---
layout: post
title: Dockerizing Your Python App and Mining Text from PDFs Along the Way
---

<img align="center" src="http://mikebirdgeneau.com/theme/images/docker_python.jpg">

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

# Deploying Python Script on the Client's Machine
Now came the tricky part. How do I make sure everything runs smoothly on the other end and the script works exactly as it did on my machine?  

In order for the transition to go smoothly, I stayed in a chatroom with my client and walked him through the execution process. At first I thought hosting my code privately on BitBucket and inviting him to be able to clone it would be a part of the solution. Unfortunately, the client did not have GIT set up, so he had to manually download the script every time I made changes to it. And changes did have to be made, as we soon discovered few quirks due to his files not always being structured the same way. I.e. what PyPDF2 saw did not always have the same RegEx pattern, so I had to broaden my patterns' scope. This resulted in many back and forths with the client. On top of that, for some reason his `pandas` module also needed to have another `openpyxl` module installed in order to work, something that was not the case on my own machine.  

Needless to say, this was not a hassle-free course to take, so I decided to simplify things for my client as well as my future projects by looking into containerization via Docker.

# Dockerizing Your Python Script (Finally!)
Long story short, I followed [instructions at runnable](https://runnable.com/docker/python/dockerize-your-python-application) to dockerize my app. My specific implementation was as folows:  

The folder where I had my `main.py` script became my Docker-building folder. Every Docker build needs a special file called `Dockerfile` (notice no file extension, it is important!) that tells Docker how to proceed with the build. The lines contained in that file are pretty self-explanatory:

```Dockerfile
# Use an official Python runtime as a parent image
FROM python:3

# Copy the current directory contents into the container at /
ADD main.py /

# Install any needed packages
RUN pip install pandas pypdf2 openpyxl

# Define environment variable
ENV NAME PDFExtract

# Run main.py when the container launches
CMD ["python", "./main.py"]
```
The breakdown of Dockerfile commands is below:
- `FROM` tells Docker which image you base your image on (in the example, Python 3).
- `ADD` tells Docker which files to include in the build and where to place them within the container.
- `RUN` tells Docker which additional commands to execute.
- `ENV` just gives a name to the Docker image.
- `CMD` tells Docker to execute the command when the image loads.

Speaking of images, I think the best way to think of the difference between them and containers is containers are instantiated images.

Once `Dockerfile` was created, I proceeded with running the following command in Terminal to build an image:
```bash
docker build -t pdfextract .
```
After the build is done, the image can be run directly on your machine. All I had to do was run the following Terminal command:
```bash
docker run -v $(pwd):/data pdfextract
```
Notice `-v $(pwd):/data` portion. There, I tell Docker to include the current PDF-containing directory in the container under `/data` folder. Take a look at my `main.py` script, it had `path = 'data/'` for the path to PDFs, so I had to include it. Naturally, if your Python app is self-contained and doesn't need anything else, you can just run `docker run image-name` Terminal command.

# Deploying Docker Container on Your Client's Machine
This is all nice and good, but how do I transfer this image to my client's computer? In order to accomplish this task, I needed to use some online repository that would store Docker images. Docker provides Docker Hub for this task. In order to host your image on Docker Hub, I followed [instructions on the official Docker site](https://docs.docker.com/get-started/part2/#share-your-image). In my case, I did as described below:  

I already had an account on [cloud.docker.com](cloud.docker.com), so all I had to do to log into it was run `docker login` in the Terminal. After I provided my username and password, I was ready to push my image to Docker Hub. But first, the image needed to be properly tagged. Even though this step is optional, it is recommended, since it is the mechanism that registries use to give Docker images a version. Tagging is easy:
```bash
docker tag pdfextract my-username/my-clients-project:a-good-descriptive-tag
```
To upload this newly created and tagged image, I ran
```bash
docker push my-username/my-clients-project:a-good-descriptive-tag
```
And that's it! The image is now hosted on Docker Hub and your client can easily run it simply by entering this Terminal command:
```bash
docker run username/project:tag
```
In my case, this was a little bit more complicated since I had to include the PDF-containing folder in the Terminal line. I instructed my client to navigate to the right folder via Terminal and then execute the following line:
```bash
docker run -v $(pwd):/data -it username/project:tag
```
# Conclusion
By dockerizing my Python app, I was able to produce the right environment consistently no matter the machine it is run on, so the script was executed correctly and predictably every time. This way, guiding my client during the deployment was minimal.

---
layout: post
title: ListServ's and Python Regular Expressions
tags: [python,regular expressions, email]
date: 2013-03-17
post_author: Seamus Tuohy
lang: en
---

List-servs are important for digital communities. Their archives form a repository of their communicative history. Luckily, the medium of e-mail adds some structure to the otherwise messy medium of human communication. I will be exploring how to take the text based list-serv archive and derive some understanding about the community behind it.

<!--more-->

This post will explore how I took plain text downloads of a list serv archive and used regular expressions to create a data structure that is easier to parse through. I will be using the language python because I find it a clear language to teach in. For newer programmers who may be reading this post I should point out that this can be done in many languages. I, for instance, did the first pass of this work in bash shell scripting, which I would not recommend.

Before we start we need to get some test data from a list-serv archive.  A list-serv archive page contains a set of text links for download. For this I used the command line tool wget. (Don't type the $)

{% highlight python %}
    $ wget -r -A.txt URL_GOES_HERE
{% endhighlight %}

This recursively grabs all files with the file extension listed behind the A from the url specified. Since I was pulling the text files from a mailman archive I just put in the mailman address and let it grab me all the text. The listserv I used lets you download the plain text directly, but some only offer gzip or other compressed formats for their files.  This will require you to do some post-processing on the files to uncompress them into plain text before playing with them.

I ran these tests on one month of two different list-serv archives in order to cross check across list-serv formatting, while keeping my testing data low.

The first thing we need to explore is the header of an e-mail. I opened up one of the text files in an editor to explore what the headers looked like. They follow the format below.

{% highlight python %}
    From person at site.blank.edu  Tue Dec  1 01:10:28 2009
    From: person at site.blank.edu (bob smith)
    Date: Tue, 12 Dec 2002 11:20:28 -0540
    Subject: [ListServ] I am a subject line
    In-Reply-To: <22234182.1259452773827.someMail.rooted@g56>
    References:
            <22234182.1259452773827.someMail.rooted@g56>
    Message-ID: <4B14DFF4.8080322G01@site.blank.edu>
{% endhighlight %}

Once I saw this I opened python through the command line and started to play around. I kept notes as I worked to walk you through the thinking process I took in exploring the data. This means that the perspective will change, and you will see me fumble, fail, and yell at my code throughout the rest of this post. If you want to, you can download an archive text file and follow along. As I make mistakes, you will see them pop out right before I start ALL_CAPS-ing all over the place. I hope it will be as fun for you as it was for me.

First, Let's open the file and create a raw text version we can manipulate. When I use "&gt;&gt;&gt;" at the begging of a line I am showing you code I ran in python. Sometimes I will use a single or double &gt; when I am showing replies in an e-mail, but I will always let you know.

{% highlight python %}
    >>> f = open("/home/name/libtech/mailman.stanford.edu/pipermail/liberationtech/2009-December.txt")
    >>> raw = f.read()
{% endhighlight %}

Now we import our regular expressions (magic) to begin to parse the text. We will be using regular expressions heavily throughout this, so it is important for you to understand generally what they are. So, go to wikipedia and read the first bit there if you don't understand. "re" is the name of the python regular expression package, so we import that first.

{% highlight python %}
    >>> import re
{% endhighlight %}

Lets find all the chunks of text that match the first line of a message like this

"From bob at the.place.edu Tue Nov 3 02:10:28 2009"

We can do this by capturing any text that starts with 'From', has some amount of other text, and is ended by four digits.

{% highlight python %}
    >>> messages = re.findall('Find.*\d\d\d\d', raw)
    >>> for i in(messages): print(i)
{% endhighlight %}

Let's break that down into its component parts.

"messages = " - This is what we want to store our regular expression as "re.findall(...)" - This is the python "re" packages way of finding all of the regular expressions that match a chunk of raw text "Find" - this looks for the text "Find" ".*" - This looks for any charicter (.) as many times in a row as it happens (*) "\d\d\d\d" - This looks for number charicters (\d) in a row ", raw" - The comma separates the regular expression before it from the raw text we created before so that re.findall knows what to run the regular expression against.

That is the last time I will go into that much detail about a command. A few helpful things for those unfamiliar with python or programming who may have difficulty following along. Any command that follows re, like re.COMMAND, is a subcommand of the re module. This is how python knows where to look for the command. Anything before an equals sign in being created, anything after is creating it.

This first attempt was a very sloppy regular expression. Lets see if we can firm it up a bit? To do this I have added the match this expression "only at the beginning" (^) and the "end"($) of line regular expression characters. To use these characters I have to use the MULTILINE flag in my re.findall command. MULTILINE runs regular expressions on only one line at a time so that we can parse the beginning of each line. (New coders, notice how I have used an equal sign later in my code to assign re.MULTILINE to the flags variable.)

{% highlight python %}
    >>>names = re.findall('^From.* \d\d\d\d$', raw, flags=re.MULTILINE)
{% endhighlight %}

We now have the first line of every message. Lets split this into its component parts. To do this we will have to see what is "regular" about our expressions and take advantage of it.

From person at site.blank.edu Tue Dec 1 01:10:28 2009

The long form regular format of these e-mails looks like this:

{% highlight python %}
    "From"[space][user name][space]"at"[space][domain name][space][day][space][month][number][space][number][colon][number][colon][number][space][number]
{% endhighlight %}

The e-mail address looks like

{% highlight python %}
    [user name][space]"at"[space][domain name].
{% endhighlight %}

The date looks like

{% highlight python %}
    [day][space][month][number][space][number][colon][number][colon][number][space][number].
{% endhighlight %}

For my purposes I care about who sent what, and when they sent it.  Because I just need to see the messages in relation to each other, and don't care about the human readable version of the date (at this point) I am going to cut that out. To specify the components of the regular expression I want returned to me I surround those expressions in parenthesis (). This allows for me to use regular expressions to match patterns across a larger set of data and only get back the things I care about. It also lets me get back the multiple pieces separately.

{% highlight python %}
    >>> names = re.findall('^From\s(.*\sat\s.*)\s*([A-Z][a-z]{2}\s[A-Z][a-z]{2}\s\d.*\d{4}$)', raw, flags=re.MULTILINE)
{% endhighlight %}

OK, I lied. Here is a quick overview of the new commands. Though for a real explanation of how they work you should check out a regular expression tutorial.

[] - This allows you to set a range for what the character will be. I use [A-Z] to say all capital letters and [a-z] to say all lower case. {} - this lets you specify a specific number of time the last character will occur. a{2} means "aa". [1-2]{2} means either "11", "12", "21", or "22" \s- this is the space character " "

But now we are getting to crazy long regular expressions. While, This regular expression is technically correct, I like to split mine up into easy to use chunks in order to make my code more readable.

{% highlight python %}
    >>> who = '(.*\sat\s.*)'
    >>> headerFront = '^From\s' + who + '\s*'
    >>> day = '[A-Z][a-z]{2}'
    >>> month = day
    >>> date = '(' + day + '\s' + month + '\s\d.*\d{4}$)'
    >>> topHeader = headerFront + date
    >>> nameNdate = re.findall(topHeader, raw, flags=re.MULTILINE)
{% endhighlight %}

Ahhhh, that's better. You will notice that I kept the text captures in the named regular expressions so that I only took the data I wanted.  This way I can reuse the who value later when I am parsing through the file.

Since I have the first line parsed the way I want it, now it is time to start grabbing text in relation to that first line. When we were only searching for one line it was nice to have the ability to use ^ and $ to identify the beginning and end of the line. Because we will be working across lines I am going to remove that in order to have our any-character expression "." match the newline character "\n" as well.  To do this I will use the DOTALL flag with re.DOTALL.

First I will replace my ^ and $ chars with \n on the regex I have.

{% highlight python %}
    >>> headerFront = '\nFrom\s' + who + '\s*'
    >>> date = '(' + day + '\s' + month + '\s\d.*\d{4})\n'
{% endhighlight %}

Now I will change my flags to include dots matching all (hint: You can also use re.S to do the same thing. re.DOTALL is just easier to read and understand a weekend or two later when I have to re-read all my code to know why that thing that should work keeps breaking.)

{% highlight python %}
    >>> nameNdate = re.findall(topHeader, raw, flags=re.DOTALL)
{% endhighlight %}

EVERYTHING IS RUINED! It captured everything... Oh yea, with periods capturing everything there are a bunch of new interesting results. Lets refine our regular expressions to really focus down what we want.

First we will replace periods that we don't want matching new lines with "§" (this is a capitalized s) which matches all non white-space characters. I am also going to remove the built-in captures "(captured text)" so that I can define my captures when I call the function. This will allow me to specify what I want to capture in the future.

{% highlight python %}
    >>> who = '\S*\sat\s\S*'
    >>> date = day + '\s' + month + '\s{2}\d*\s\S*?\d{4}$'
{% endhighlight %}

With that I can construct a regular expression parser that will let me create a collection that presents me with who an e-mail to a list is from, what date it was sent, and then the contents (including the rest of the header info I have not collected yet)

{% highlight python %}
    >>> who = '\S*\sat\s\S*'
    >>> headerFront = '^From\s' + who + '\s*'
    >>> capturedFront = >>> headerFront = '^From\s(' + who + ')\s*'
    >>> day = '[A-Z][a-z]{2}'
    >>> month = day
    >>> date = day + '\s' + month + '\s{2}\d*\s\S*?\d{4}$'
    >>> capHeader = capturedFront + '(' date ')'
    >>> dropHeader = headerFront + date

    >>>emailList = re.findall(capHeader + '(.*?)' + dropHeader, raw, flags=re.DOTALL)
{% endhighlight %}

or testing purposes I have taken all of the single lines of text and put them in a set of functions to make it easier to quickly modify and check changes in the code. This is that moment I have been dreading. I have switched from playing around on the command line to writing down code that I will have to maintain, comment, document, test, etc. This next paragraph is for new coders, so anyone who is still reading this, who has a strong grasp of python and coding can skip ahead to where I all caps you to start reading again.

By putting everything in chunk of code I can call all of my setup functions to begin with. So, I start by importing my dependencies "re" and creating a class to hold all the command within. A class is a way of separating a cope of a set of functions and values into a cohesive unit that will not interfere with other nearby units that use the same functions and create similarly names values. I decided to wrap all my functions in a class so that I will be able to manipulate multiple listServ files on the command line independently and simultaneously.

Python follows a structure where you define a function "def FUNCTION_NAME():" and then indent what belongs to that function within it. Within the parens, you can also put down commands that the function takes, separated by commas. When a function lives in a class it always takes "self" first in order to make sure that it only operates on the independent instance that it lives within.

{% highlight python %}
    def functionName(self, input)
{% endhighlight %}

The first thing you will see in a python function should be a comment about what the code does. This will be surrounded by three quotes on each side and will give you an overview of the function.

{% highlight python %}
    def functionName(self, input)
        """I am an example function. I take in an input and do nothing else at this point"""
{% endhighlight %}

Lastly, before we dig in, you will notice that there is a __init__ function. That is a class specific function that creates values upon creating a new function. I use it to define some default variables.

### CODERS START READING AGAIN HERE ------&gt;

{% highlight python %}
    import re
    class converter:
        def __init__(self):
            self.raw = ''

        def getArchive(self, textFile):
            """This function takes the location of the list-serv text file and opens it up for parsing. Much later I may add the ability to just choose the html address of a list-serv archive. That will be straight up neato!
            """
            text = open(textFile)
            rawText = text.read()
            self.raw = rawText


        def printMessage(self):
            """ This function parses an archive and prints out the results of a generic regular expression. For testing purposes only. Will be converted into a generic dictionary generator that parses the text-file.
    """
            if self.raw == ''
               print("Please get a list serv archive and import it file first.")
            who = '\S*\sat\s\S*'
            headerFront = '\nFrom\s' + who + '\s*'
            capturedFront = '\nFrom\s(' + who + ')\s*'
            day = '[A-Z][a-z]{2}'
            month = day
            date = day + '\s' + month + '\s*?\d*?\s\S*?\s\d{4}\n'
            capHeader = capturedFront + '(' + date + ')'
            dropHeader = headerFront + date
    #TODO - create captures for all header sections
    #TODO - rewrite the following line to become a dictionary that parses the monthly log and create individual dictionaries of all pertinent header info for each e-mail and includes the content.
            messageDict = re.findall(capHeader, self.raw, flags=re.DOTALL)
            print messageDict
{% endhighlight %}

Now that I have my quick regular expression parser I can just change the messageDict to examine my progress. I created a small one liner to reload my function, create a class object, run the function that grabs the text, and run a test print of my regular expression. I saved this function as "main.py" in the same directory as "testText", which is a downloaded listServ archive. From within the directory of those files I re-ran python and then the following command.

{% highlight python %}
    >>> reload(main); a=main.converter(); a.getArchive('testText'); a.printMessage()
{% endhighlight %}

The semicolons allow me to run multiple commands on the same line. This is nice because I can use the arrow up key to move back in my python history to re-load the whole code block and test changes I made in my code.

Now the next step is to capture the rest of the information and put it in a format that is easy to manipulate. To do this we are going to create a dictionary from the parsed data. Since you have a good understanding of Regular Expressions now I will try to only go in-depth when discussing new concepts. Here is our header again.

{% highlight python %}
    From person at site.blank.edu  Tue Dec  1 01:10:28 2009
    From: person at site.blank.edu (bob smith)
    Date: Tue, 12 Dec 2002 11:20:28 -0540
    Subject: [ListServ] I am a subject line
    In-Reply-To: <22234182.1259452773827.someMail.rooted@g56>
    References:
            <22234182.1259452773827.someMail.rooted@g56>
    Message-ID: <4B14DFF4.8080322G01@site.blank.edu>
{% endhighlight %}

At first glance it is easy enough to parse out the contents of the header using a series of regular expressions for each line such as this.

{% highlight python %}
    >>> whom = 'From\:\s(.*?)\n'
    >>> date = 'Date\:\s(.*?)\n'
    >>> subject = 'Subject\:\s(.*?)\n'
    >>> inReply = 'In\-Reply\-To\:\s(.*?)\n'
    >>> refrences = 'Refrences\:\s(.*?)\n'
    >>> messageID = 'Message\-ID\:\s(.*?)\n'
    >>> headerChunks = from + date + subject + inReply + refrences + messageID
    >>> messageDict = re.findall(headerChunks, self.raw, flags=re.DOTALL)
{% endhighlight %}

This won't get all the data. Even worse, it will be all sorts of slow.  In fact, I did that set of regular expressions just for your benefit.  Not saying that you owe me or anything. I am just saying. So, why won't that work? Think back to your past e-mails, look at our example header, and think about what re.findall does and returns. It is not because I am not grabbing the surrounding text.

If you figured out that not every message is in reply to something else you were right on. The first e-mail in a archive will not have this section. References are the same way. So, we have to come up with a better way to deal with these cases.

Since we know that a header will always have our archive specific front matter, which is repeated in the header, and a message ID we can safely identify the bounds of the header and grab just it.

{% highlight python %}
    >>> headerFront = '\nFrom\s' + who + '\s*'
    >>> day = '[A-Z][a-z]{2}'
    >>> month = day
    >>> date = day + '\s' + month + '\s*?\d*?\s\S*?\s\d{4}\n'
    >>> dropTop = headerFront + date
    >>> getHeader = '(.*?Message\-ID\:\s.*?\n)'
    >>> messageDict = re.findall(dropTop + getHeader, self.raw, flags=re.DOTALL)
{% endhighlight %}

Now we have a function that only grabs the header of an e-mail from a list-serv. But, we are also dropping the archive specific front matter that we don't like. Lets use this and pythons regular expression "split" function to parse our e-mails out a bit further.

{% highlight python %}
            splitText = re.split(dropTop, self.raw)
{% endhighlight %}

Using the split function we have created a list of e-mails. This does everything we need to correctly parse an e-mail archive. That is, of course, unless someone has the nerve to copy and paste a list-serv message into the text of an e-mail. If that happened, the e-mail would be split up. For instance, if I pasted the text of this article into an e-mail to a listServ it would match a whole bunch of stuff that it is not supposed to and, therefore, cut it into ribbons and ruin any chance of me parsing that listServ correctly. Just warning you, edge cases...  they're evil.

The next step will be to fully parse the headers of a message. Remember when I made you feel all bad about how much extra work I did just to show you what those regular expressions for each line would look like.  That was a lie. I am going to use those right now. Let that be a lesson to you. Humans are mischievous little devils and should not be trusted.  But, on to the lesson.

Now that we have split apart our e-mails we can easily pass them to a function that identifies the header sections and splits them apart. In order to make the next section easier, I am also going to use this function to create a dictionary out of split up components. New coders should note that the "#" character comments out single lines of comments in python code.

{% highlight python %}
        def dictify(self, email):
            #get headers from email
            getHeader = '(.*?Message\-ID\:\s.*?\n)'
            msgDict = {}
            msg = re.findall(getHeader + '(.*)', email, flags=re.DOTALL)
            #create a dictionary item for body text
            for i in msg:
                msgDict['body'] = i[1]
            # Setting header specific regEx's
            whom = 'From\:\s(.*?)\n'
            date = 'Date\:\s(.*?)\n'
            subject = 'Subject\:\s(.*?)\n'
            inReply = 'In\-Reply\-To\:\s(.*?)\n'
            references = 'References\:\s(.*?)\nMessage\-ID\:'
            messageID = 'Message\-ID\:\s(.*?)\n'

            #create a dictionary item for the header items that are always there.
            msgDict['From'] = re.findall(whom, email, flags=re.DOTALL)
            msgDict['Date'] = re.findall(date, email, flags=re.DOTALL)
            msgDict['Subject'] = re.findall(subject, email, flags=re.DOTALL)
            msgDict['ID'] = re.findall(messageID, email, flags=re.DOTALL)

            #create checks for items that may not be there.
            if re.search(references, email, flags=re.DOTALL) != 'none':
                msgDict['References'] = re.findall(references, email, flags=re.DOTALL)
            if re.search(inReply, email, flags=re.DOTALL) != 'none':
                msgDict['Reply'] = re.findall(inReply, email, flags=re.DOTALL)
            #split up refrences into a list within its dict item for easy parsing later
            if msgDict['References'] != []:
                msgDict['References'] = re.split('\s*|\n\t', msgDict['References'][0])

            #remember we are just printing out sections to check for consistancy. I created a simple 'Refrences' to ID printout here to look at how a reply links to the refrences before it.
            print("--------------------------")
            print("=======NEW EMAIL=========")
            print("--------------------------")
            print("==========ID==============")
            print(msgDict['ID'])
            print("==========Reply To==============")
            print(msgDict['Reply'])
            print("=========Refrences==========")
            print(msgDict['References'])
{% endhighlight %}

When we are done we get a dictionary called messages that holds each e-mail as an item that looks like this. New coders, brackets are used to hold items in a dictionary. Within brackets the dictionary has keys and items as such: [key:item, key2:item2] You will notice that in my parsing I have made it so that each item is contained in a smaller list.  This is going to cause me all sorts of problems later, but we will ignore that for now.

{% highlight python %}
    ['Body': ["ALL SORTS OF BODY TEXT"],
     'From': ['bob at ILovePeanuts.com (Bob Peanut)'],
     'Refrences': ['<50E0B2B7.7010304@butterOfTheNut.org>'],
     'Date': ['Mon, 01 Dec 2002 18:09:37 -0600'],
     'Reply': ['<50E0B2B7.7010304@butterOfTheNut.org>'],
     'ID': ['<07080613-2615-4B02-BD9D-8024FD1CD62E@designafternext.com>', '<50E0B2B7.7010304@butterOfTheNut.org>'],
     'Subject': ["[LegumeLovers] Macadamia nuts are for the weak!"]
{% endhighlight %}

Well, that is it for parsing the e-mail headers. In the next piece I will go in more depth about how we parse the body text to make it easier to connect in-line responses to their initial message and how we refactor our hacked up code to better expose our list-serv parsers API for others to use it for content analysis.

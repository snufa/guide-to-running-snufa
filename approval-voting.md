(voting)=
# Approval voting

The aim of this notebook is to show some code for generating a Qualtrics voting form for a workshop, as we used in [SNUFA 2022](https://snufa.net/2022).

## Code

You can use [](./code/generate_survey.ipynb) to generate a nice HTML representation of the code and the survey file to import into Qualtrics.

You can use [](./code/generate_abstracts.ipynb) to generate the markdown files for abstracts to upload to the conference website.

## Step 1: Collect abstracts

We used [Office Forms](https://forms.office.com) but Google forms or any other system would work as well. We collected author names, emails, abstract text and whether or not you wanted to be considered for poster or talk. For our precise list of questions, see @submissions. If you change the questions, you'll need to tweak the code below slightly.

From this we get an Excel file that can be converted to a CSV. Make sure to use the option to save as CSV-UTF8 since you'll probably have accents in some names.

## Step 2: Generate Qualtrics survey

### Qualtrics import file format

[Qualtrics](https://www.qualtrics.com) is a survey tool. I used it because Imperial College has a site license and because it has some nice features for importing forms, which I'll use below.

The main step is generating a text file that can be used to import questions using [Qualtric's somewhat unclearly documented "advanced format"](https://www.qualtrics.com/support/survey-platform/survey-module/survey-tools/import-and-export-surveys/). These look a bit like this:

```
[[AdvancedFormat]]

[[Question:DB]]

<div>
    <h3>Presentation title</h3>
    <p style="font-size: 80%;">
        The text of the abstract here.
    </p>
</div>

[[Question:MC:SingleAnswer]]
[[ID:abstract1yesno]]
I would like to see this as a talk.
[[Choices]]
No
Yes

[[Question:TextEntry:Form]]
[[ID:abstract1comments]
Any comments?
[[Choices]]
Insert comments here.
```

Once you have a file in this format, you can [import it into Qualtrics](https://www.qualtrics.com/support/survey-platform/survey-module/survey-tools/import-and-export-surveys/).

### Using Python and Jinja to generate the form

We use Python and the [Jinja templating engine](https://palletsprojects.com/p/jinja/) to do this. In addition to the above, we insert a ``[[Block]]`` in between each batch of questions. Here's the code.

```Python
# Import a few standard packages
import csv
import jinja2
from IPython.core.display import HTML

# Read the submissions and do a bit of data cleaning
with open('submissions.csv', newline='', encoding='utf8') as csvfile:
    csvreader = csv.DictReader(csvfile, dialect='excel')
    submissions = list(csvreader)
for submission in submissions:
    for k, v in list(submission.items()):
        # This character shows up for some reason (unclear)
        k_new = k.replace('\ufeff', '').strip().split('\n')[0]
        # Some people use all caps for the title, so we automatically fix that
        if k_new=='Presentation title' and v.upper()==v:
            v = v.title()
        # some people paste text with newlines for every line which looks ugly, so we detect that and automatically fix
        if k_new=='Abstract (please keep under 300 words)' and max(map(len, v.split('\n')))<120:
            v = v.replace('\n', ' ')
        submission[k_new] = v
    submission['ID'] = submission['\ufeffID'] # not sure why forms inserts this random character

# This generates the survey text that can be imported into Qualtrics
template = jinja2.Template('''
[[AdvancedFormat]]
    
{% for sub in submissions %}

[[Block]]

[[Question:DB]]
    
<div>
    <h3>{{ sub['Presentation title'] }}</h3>
    {% for para in sub['Abstract (please keep under 300 words)'].splitlines() %}
    {% if para.strip() %}
    <p style="font-size: 80%;">
        {{ para }}
    </p>
    {% endif %}
    {% endfor %}
</div>

[[Question:MC:SingleAnswer]]
[[ID:abstract{{ sub.ID }}yesno]]
I would like to see this as a talk.
[[Choices]]
No
Yes

[[Question:TextEntry:Form]]
[[ID:abstract{{ sub.ID }}comments]
Any comments?
[[Choices]]
Insert comments here.
{% endfor %}
''')

survey = template.render(submissions=submissions)

#HTML(template.render(submissions=submissions))
print(survey)
open('survey.txt', 'w', encoding='utf-8').write(survey)
```

### Importing and tweaking the form in Qualtrics

Having imported the generated ``survey.txt`` file into a new Qualtrics survey, I added randomisation. Asking your audience to vote on all the abstracts might be a bit too much, so we can show each voter a random subset using [Qualtrics' Randomizer](https://www.qualtrics.com/support/survey-platform/survey-module/survey-flow/standard-elements/randomizer/). Create a Randomizer in the survey flow, drag all the imported blocks into it, and ask it to show 10 at a time with the "Evenly Present Elements" option. Note: this value worked well for SNUFA 2023 for example, where we had about 40 abstracts and 160 people participated in the voting (of around 700 who registered), meaning that we got about 40 votes per abstract. If your abstract to participant ratio is different, you might want to increase the number of abstracts each person has to evaluate.

You might also want to tweak the options for how the form is displayed, add a logo, intro page, etc.

## Step 3: Publicise and collect votes

Send a copy to all your workshop participants and give them a week to respond.

## Step 4: Analyse data

Download the votes in CSV format. Load them with something like this:

```Python
with open('votes.csv', newline='') as csvfile:
    csvreader = csv.DictReader(csvfile, dialect='excel')
    votes = list(csvreader)

votes = votes[2:]

print(f'{len(votes)} raw votes')

ip_address_counts = defaultdict(int)
for vote in votes:
    ip_address_counts[vote['IPAddress']] += 1

votes = [vote for vote in votes if ip_address_counts[vote['IPAddress']]==1]
```

Note that Qualtrics keeps a copy of the IP address for each submission, and we delete any votes if more than one vote came from that address. For a larger event you might need a more sophisticated strategy than this, e.g. tying votes to unique registration.

With the setup above, we can store the number of yes/no votes for each abstract as follows.

```Python
id_to_submission = {}
for submission in submissions:
    submission['yes_votes'] = 0
    submission['no_votes'] = 0
    submission['comments'] = []
    id_to_submission[submission['ID']] = submission
    
for vote in votes:
    vote['yes'] = yes_votes = []
    vote['no'] = no_votes = []
    for k, v in vote.items():
        if v:
            if k.startswith('abstract'):
                k = k.replace('abstract', '')
                if k.endswith('yesno'):
                    k = k.replace('yesno', '')
                    if v=='Yes':
                        id_to_submission[k]['yes_votes'] += 1
                        yes_votes.append(k)
                    elif v=='No':
                        id_to_submission[k]['no_votes'] += 1
                        no_votes.append(k)
                elif k.endswith('comment_1'):
                    k = k.replace('comment_1', '')
                    id_to_submission[k]['comments'].append(v)
```

Now we can order by "approval" (ratio of yes to no votes).

```Python
poster_submissions = [submission for submission in submissions if submission['Would you prefer a poster or talk (if selected)?']!="Talk preferred"]
talk_submissions = [submission for submission in submissions if submission['Would you prefer a poster or talk (if selected)?']=="Talk preferred"]
talk_submissions.sort(reverse=True, key=lambda submission: submission['yes_votes']/(submission['yes_votes']+submission['no_votes']))

for sub in talk_submissions:
    sub['approval'] = sub['yes_votes']/(sub['yes_votes']+sub['no_votes'])
```

And generate a nice HTML file for organisers to see what's going on.

```Python
template = jinja2.Template('''
<html><head><title>Submissions</title></head><body>
    
    <h1>Talk preferred</h1>
    {% for sub in talk_submissions %}
        <h3>{{ sub['Presentation title'] }}</h3>
        <h4>{{ sub['Presentation authors'] }}</h4>
        <h4>Corresponding author: <a href="mailto:{{ sub['Corresponding author email address'] }}">{{ sub['Corresponding author name'] }}</a></h4>
        {% for para in sub['Abstract (please keep under 300 words)'].splitlines() %}
            {% if para.strip() %}
                <p>
                    {{ para }}
                </p>
            {% endif %}
        {% endfor %}
        <p>
            <span style="background: lightgreen; border-radius: 10px; padding: 10px; display: inline-block; margin: 1px;">
                👍 <b>{{ sub['yes_votes'] }}</b> yes
            </span>
            <span style="background: lightpink; border-radius: 10px; padding: 10px; display: inline-block; margin: 1px;">
                👎 <b>{{ sub['no_votes'] }}</b> no
            </span>
            <span style="background: lightblue; border-radius: 10px; padding: 10px; display: inline-block; margin: 1px;">
                <b>{{ int(100*sub['yes_votes']/(sub['yes_votes']+sub['no_votes'])) }}%</b> positive
            </span>            
        </p>
        {% if sub['comments'] %}
        <p>
            Comments:
        </p>
        <ul>
            {% for comment in sub['comments'] %}
                <li>
                    {{ comment }}
                </li>
            {% endfor %}
        </ul>
        {% endif %}
    {% endfor %}

    <h1>Poster preferred</h1>
    {% for sub in poster_submissions %}
        <h3>{{ sub['Presentation title'] }}</h3>
        <h4>{{ sub['Presentation authors'] }}</h4>
        {% for para in sub['Abstract (please keep under 300 words)'].splitlines() %}
            {% if para.strip() %}
                <p>
                    {{ para }}
                </p>
            {% endif %}
        {% endfor %}
    {% endfor %}
</body></html>
''')

submissions_html = template.render(talk_submissions=talk_submissions, poster_submissions=poster_submissions, int=int)

open('submissions_with_votes.html', 'w', encoding='utf-8').write(submissions_html)
```

## Step 5: Finalise the programme

From this list, you can pick the top N as talks, flash talks, etc. In SNUFA 2024 we took the top 7 as short talks and the next 10 as two minute flash talks. We save this into a new CSV file ``selected.csv``. Email the authors and confirm times, etc. You may need to remove some as authors will withdraw at this point, and you can promote some flash talks to full talks, and posters to flash talks, and so on. We store this in ``selected.csv`` with a column ``decision``.

Once you have all the confirmations, you can generate abstracts as markdown files and upload to the website, see [](./code/generate_abstracts.ipynb).
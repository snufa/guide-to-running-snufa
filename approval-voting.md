(voting)=
# Approval voting

```{warning}
This guide needs a bit of updating.
```

The aim of this notebook is to show some code for generating a Qualtrics voting form for a workshop, as we used in [SNUFA 2022](https://snufa.net/2022).

## Step 1: Collect abstracts

We used [Office Forms](https://forms.office.com) but Google forms or any other system would work as well. We collected author names, emails, abstract text and whether or not you wanted to be considered for poster or talk.

From this we get an Excel file that can be converted to a CSV.

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
import csv
import jinja2

with open('submissions.csv', newline='') as csvfile:
    csvreader = csv.DictReader(csvfile, dialect='excel')
    submissions = list(csvreader)

template = jinja2.Template('''
[[AdvancedFormat]]
    
{% for sub in submissions %}
{% if sub['Would you prefer a poster or talk (if selected)?']=="Talk preferred" %}

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
{% endif %}
{% endfor %}
''')

survey = template.render(submissions=submissions)

open('survey.txt', 'w', encoding='utf-8').write(survey)
```

### Importing and tweaking the form in Qualtrics

Having imported the generated ``survey.txt`` file into a new Qualtrics survey, I added randomisation. Asking your audience to vote on all the abstracts might be a bit too much, so we can show each voter a random subset using [Qualtrics' Randomizer](https://www.qualtrics.com/support/survey-platform/survey-module/survey-flow/standard-elements/randomizer/). Create a Randomizer in the survey flow, drag all the imported blocks into it, and ask it to show 10 at a time with the "Evenly Present Elements" option.

You might also want to tweak the options for how the form is displayed, add a logo, intro page, etc.

## Step 3: Publicise and collect votes

Send a copy to all your workshop participants and give them a week to respond.

## Step 4: Analyse your data

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

We take the top 8 in our case, and get a list of email addresses to let authors know.

```Python
all_talks = talk_submissions[:8]
all_posters = talk_submissions[8:]+poster_submissions
for sub in all_talks:
    sub['talk'] = True
for sub in all_posters:
    sub['talk'] = False
    
# Talk emails
print("Talk emails:", ', '.join(sub['Corresponding author email address'] for sub in talk_submissions[:8]))

# Poster emails
all_posters = talk_submissions[8:]+poster_submissions
all_posters.sort(key=lambda sub: sub['Corresponding author name'])
print("Poster emails: ", ', '.join(sub['Corresponding author email address'] for sub in all_posters))
```

We might want to take a look at the histogram of votes.

```Python
boundary = (talk_submissions[8]['approval']+talk_submissions[7]['approval'])/2
print(f'Boundary = {round(100*boundary)}%')
binedges = boundary+np.arange(-20, 21)*0.1
counts, binedges, _ = plt.hist([sub['approval'] for sub in talk_submissions], bins=binedges, label='All submissions (talk preferred)')
plt.hist([sub['approval'] for sub in talk_submissions[:8]], bins=binedges, label='Accepted for talk')
plt.axvline(boundary, ls='--', c='k', label=f'Cutoff = {round(100*boundary)}%')
plt.xlabel('Fraction interested in seeing submission as talk')
plt.ylabel('Number of submissions')
plt.xlim(0, 1)
plt.legend(loc='best')
plt.tight_layout()
```

And finally generate an abstracts list.

```Python
def generate_stub(sub):
    title_words = sub['Presentation title'].split(' ')
    title_words = [word for word in title_words if len(word)>3]
    name = ' '.join(sub['Corresponding author name'].split(' ')[:2])
    stub = name+' '+title_words[0]
    stub = stub.lower().replace(' ', '-').replace(':', '').replace('.', '').replace(',', '')
    return stub

from random import shuffle
shuffle(submissions)

template = jinja2.Template('''
# {{ sub['Presentation title'] }}

**Authors:** {{ sub['Presentation authors'] }}

**Presentation type:** {{ 'Talk' if sub['talk'] else 'Poster' }}

## Abstract

{{ sub['Abstract (please keep under 300 words)'] }}
''')

for sub in submissions:
    stubname = generate_stub(sub)
    sub['abstract_filename'] = fname = f'abstracts/{stubname}.md'
    with open(fname, 'w', encoding='utf-8') as f:
        f.write(template.render(sub=sub))

template = jinja2.Template('''
# SNUFA 2022 Abstracts

## Invited talks (TBD)

## Contributed talks

{% for sub in all_talks | sort(attribute='abstract_filename') %}
* [{{ sub['Presentation title'] }}]({{ sub['abstract_filename'] }}) ({{ sub['Presentation authors'] }})
{% endfor %}

## Posters

{% for sub in all_posters | sort(attribute='abstract_filename') %}
* [{{ sub['Presentation title'] }}]({{ sub['abstract_filename'] }}) ({{ sub['Presentation authors'] }})
{% endfor %}

''')

open('all_abstracts.md', 'w', encoding='utf-8').write(template.render(all_talks=all_talks, all_posters=all_posters))
```

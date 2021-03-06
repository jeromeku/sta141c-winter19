Topics:

- Siteng has office hours today at 2pm in the conference room
- usaspending.gov has agreed to feature 1-2 student projects at the [Student Innovator's page](https://datalab.usaspending.gov/student-innovators-toolbox.html), subject to approval by External Affairs
- Address feedback from mid quarter inquiry
- Review bash scripts, cluster submission, and code style
- Looping in bash to srun
- ? Integrate R and Python scripts into pipelines

Reading:

- [SWC: Unix shell lesson](https://swcarpentry.github.io/shell-novice/)
- [Wikipedia: Anti Patterns](https://en.wikipedia.org/wiki/Anti-pattern#Software_engineering)
- [Stack Overflow: Useless Use of `cat`](https://stackoverflow.com/a/16619430/2681019)
- [Stack Overflow: Optimizing shell scripts](https://unix.stackexchange.com/a/67131)

Terms:

- __pipe__ `cmd1 | cmd2` binary operator to redirect `stdout` from one command to another.
- __environment variable__ `FILE=transactions.zip` The shell will replace `${FILE}` with `transactions.zip` wherever the former occurs.
- __option__ aka flag, parameter, argument. In `cmd --foo`, `--foo` is the option.
    There may also be a short form, say `-f`.


## Rubric

Most students are new to bash, so I don't expect that you know what can be done.
Here's how we'll grade the code for this homework.
Let's have everyone on the same page.

Also, some aspects of how you write code really matter, while others are more a matter of taste.

The code you turn in for this class should be good- the best you can write.
I'll tell you the difference between good and bad here.
The graders will be following exactly this.

These lessons apply to most languages, not just bash.

```{bash}
# Where I keep an example data set
cd ~/projects/sta141c-winter19/data
```


#### Use variables rather than magic numbers

Magic numbers are arbitrary numbers that appear sprinkled throughout your code.
Replace them with descriptive variable names.

For example, suppose I want to select the fiscal year from the data.

`FISCAL_YEAR` is the same name as a column- great.

😀

```{bash}
FISCAL_YEAR=5
head transaction_small.csv |
    cut -d , -f ${FISCAL_YEAR}
```

I may want to just shorten it to `YEAR`.
Totally fine.

😀

```{bash}
YEAR=5
head transaction_small.csv |
    cut -d , -f ${YEAR}
```

------------------------------------------------------------

`COLUMN` is not a descriptive name.

😐

```{bash}
COLUMN=5
head transaction_small.csv |
    cut -d , -f ${COLUMN}
```

Descriptive variable names and clear code are better than comments.
__Question__ Why?
They embed the meaning into the program.
It's more in line with the DRY principle.

😐

```{bash}
head transaction_small.csv |
    cut -d , -f 5   # 5 is the 'fiscal_year' column
```

------------------------------------------------------------

5 is a magic number.
What does it mean?

😡

```{bash}
head transaction_small.csv |
    cut -d , -f 5
```

Compare the code above with the version below which is readable, but incorrect:

```{bash}
YEAR=6
head transaction_small.csv |
    cut -d , -f ${YEAR}
```

This is a typo, an innocent mistake anyone could make.
The code clearly conveys the intention.

__I value clarity over correctness.__
Therefore this code, although incorrect, is better than the unclear code above.
It's fine to write the unclear code when you're casually examining the data.
But when you turn it in, make it as good as you can.

All complex programs will have bugs.
It's a fact of life.
To reduce bugs, write clear, beautiful code.

__Question__: The 'good' solution I showed used variable names with numbers.
What would be even better?
To use the names right from the data, i.e. refer to the columns by names directly, without having to use a variable.
This is how data frames work.

We could make bash do this, but it would be more trouble than it's worth.


#### DRY

Do not repeat yourself.
Any time you see the same pattern repeated over and over in code, ask yourself, is there a better way?

The code below uses `transaction_small.csv` several times.
What happens when we want to switch it from the testing code to the actual large data set?
All we do is change the file name that the variable refers to- easy.

😀

```{bash}
DATAFILE="transaction_small.csv"
head ${DATAFILE} |
    wc -l

cat ${DATAFILE} |
    grep "National" |
    wc -l

YEAR=5
head ${DATAFILE} |
    cut -d , -f ${YEAR}
```

Here's a version where it's hardcoded.
What happens when we want to switch it from the testing code to the actual large data set?
We have to replace all instances of `transaction_small.csv` with the name of the large file- error prone.

😡

```{bash}
head transaction_small.csv |
    wc -l

cat transaction_small.csv |
    grep "National" |
    wc -l

YEAR=5
head transaction_small.csv |
    cut -d , -f ${YEAR}
```


#### Efficient

The order in which you run the commands matters, even if they produce the same result.

One way to make your programs efficient is to remove as much of the data as soon as you possibly can.
Run the filter commands first.

Sorting in particular is expensive, so you want to filter rows and columns before sorting.
Below the sort comes after the filter.

😀

```{bash}
cat transaction_small.csv |
    grep "National" |
    sort --numeric-sort
```

Here the sort comes before the filter, so it must sort more data.

😡

```{bash}
cat transaction_small.csv |
    sort --numeric-sort |
    grep "National"
```

This means multiple different commands can produce the same output, but some may be more efficient.

Side note- the astute reader may notice the 'useless use of cat', and correctly say, hey, that's not efficient!
I'm doing it for consistency and ease of development, and `cat` doesn't use that many resources.


#### Write out parameter names

Suppose you carefully read the `man` page for `sort` and you decide to use the options that let it sort based on a different column.

When we specify the arguments completely by name then most people can figure out what they do.

Question: which of the two options are more clear?

😀

```{bash}
YEAR=5
cat transaction_small.csv |
    sort --key=${YEAR} --field-separator=, --numeric-sort
```

bash has a reputation for being cryptic, and the one letter flags are one of the reasons.
They have their place for interactive use.

😡

```{bash}
YEAR=5
cat transaction_small.csv |
    sort -k ${YEAR} -t , -n
```


#### Use a consistent style

Both commands below are one liners- bash runs them in exactly the same way.
In the first we've followed the convention that every time we use a pipe we start a new line.

Which is easier to read?
Which is easier to write?
__Question:__ Why?
The first is better for both reading and writing.
Here the first part of each line is the command, so without any effort we see that it uses `cat, grep, sort`.

😀

```{bash}
YEAR=5
cat transaction_small.csv |
    grep --no-filename "National" |
    sort --key=${YEAR} --field-separator=, --numeric-sort |
    cat > result.txt
```

😡

```{bash}
YEAR=5
cat transaction_small.csv | grep --no-filename "National" | sort --key=${YEAR} --field-separator=, --numeric-sort | cat > result.txt
```

For the purposes of the bash assignment, I want you to put each piped operation on a new line.

It's a matter of personal preference to whether you put spaces in front or not.
I prefer the way I've been writing it- that's why I write it that way :)
__Question__: Can anyone think of a good reason to prefer one over the other?

😀

```{bash}
YEAR=5
cat transaction_small.csv |
grep --no-filename "National" |
sort --key=${YEAR} --field-separator=, --numeric-sort |
cat > result.txt
```

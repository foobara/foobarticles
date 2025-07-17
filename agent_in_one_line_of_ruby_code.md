## Writing an AI Agent in 1 Line of Ruby Code! (using Foobara's AgentBackedCommand)

In this article, we'll introduce/demo `Foobara::AgentBackedCommand` by adding it to a program
with an existing Foobara domain in it.

### The Demo Loan Origination Domain and Demo Records

First, we need a program to add it to. Here is a program that uses a demo loan origination domain 
written using Foobara. You can find the demo domain implementation here: 
https://github.com/foobara-demo/loan-origination/tree/main/src

So here's our program in an executable file called `loan-origination`:

```ruby
#!/usr/bin/env ruby

require_relative "boot"
require "foobara/sh_cli_connector"

connector = Foobara::CommandConnectors::ShCliConnector.new
connector.connect(FoobaraDemo::LoanOrigination)
connector.run
```

This uses a Foobara CLI command connector so that we can run commands from `LoanOrigination` on the command line.

Let's see what commands are available:

```
$ ./loan-origination 
Usage: loan-origination [GLOBAL_OPTIONS] [ACTION] [COMMAND_OR_TYPE] [COMMAND_INPUTS]
...
Available commands:
  FoobaraDemo::LoanOrigination::SubmitApplicationForUnderwriterReview
  FoobaraDemo::LoanOrigination::FindALoanFileThatNeedsReview            Will return a loan file
                                                                        that needs review or nil
                                                                        if there are no loan 
                                                                        files that need review
  FoobaraDemo::LoanOrigination::FindAllLoanFiles
  FoobaraDemo::LoanOrigination::GenerateLoanFilesReport
  FoobaraDemo::LoanOrigination::StartUnderwriterReview
  FoobaraDemo::LoanOrigination::StartLoanApplication
  FoobaraDemo::LoanOrigination::CreateCreditPolicy
  FoobaraDemo::LoanOrigination::FindCreditPolicy
  FoobaraDemo::LoanOrigination::ApproveLoanFile
  FoobaraDemo::LoanOrigination::AddCreditScore
  FoobaraDemo::LoanOrigination::FindLoanFile
  FoobaraDemo::LoanOrigination::DenyLoanFile
  FoobaraDemo::LoanOrigination::AddPayStub                              Add a pay stub to a loan
                                                                        file
  FoobaraDemo::LoanOrigination::Demo::PrepareDemoRecords
$ 
```

So we can see a ton of commands at our disposal. Let's run `GenerateLoanFilesReport`:

```
$ ./loan-origination GenerateLoanFilesReport
$ 
```

We get nothing back. Which makes sense! Let's create some records to play with using `PrepareDemoRecords`:

```
$ ./loan-origination PrepareDemoRecords
...
$ 
```

This prints out a bunch of loan file data that is excluded. Now let's look at our report again:

```
$ ./loan-origination GenerateLoanFilesReport
{
  id: 1,
  applicant: "Barbara",
  state: "needs_review"
},
{
  id: 2,
  applicant: "Basil",
  state: "needs_review"
},
{
  id: 3,
  applicant: "Fumiko",
  state: "needs_review"
}
$ 
```

This command creates 3 loan files ready for review that are all on different credit policies. 
We expect these loan files to
be denied or approved for different reasons. See `PrepareDemoRecords` for details on the demo records
and the expected results of reviewing these loan files.

Now, we could review these loan files ourselves using `loan-origination` 
since it exposes all the commands that we need. We could do the following:

```
$ ./loan-origination FindALoanFileThatNeedsReview
...
id: 1
$ 
```

We could then do `./loan-origination StartUnderwriterReview --loan-file 1` which would start our review.

We would then need to do `./loan-origination FindCreditPolicy --credit-policy 1` and finally we could
run `./loan-origination DenyLoanFile --loan-file 1 --credit-score-used 650 --denied-reasons low_credit_score`.

We could then move on to the next loan file until no more need review.

In this case, we're a human operating the domain, which we've chosen in this case to expose through a CLI.

If we had an AI agent that also knew how to operate our domain, we could say something like: 
"Hi! Could you please review every loan file that needs review? Thanks!"

Notice that what we would be doing is issuing a high-level domain operation to our AI agent.
Since Foobara commands are meant to encapsulate high-level domain operations, it would be nice if we
could just have a `ReviewAllLoanFiles` Foobara command but without having to write its #execute method (ie,
the domain logic itself.)

We can!

### Defining and Running a Foobara::AgentBackedCommand

Now it's finally time to write our AI agent in 1 line of Ruby code, introducing `Foobara::AgentBackedCommand`:

```ruby
#!/usr/bin/env ruby

require_relative "boot"
require "foobara/sh_cli_connector"

class FoobaraDemo::LoanOrigination::ReviewAllLoanFiles < Foobara::AgentBackedCommand; end

connector = Foobara::CommandConnectors::ShCliConnector.new
connector.connect(FoobaraDemo::LoanOrigination)
connector.run
```

And we're done! We've created a command called `ReviewAllLoanFiles` which is an `AgentBackedCommand`. That line
in the middle of the file.

We can now see it in our `loan-origination --help` output:

```
$ ./loan-origination --help
...
Available commands:

  FoobaraDemo::LoanOrigination::SubmitApplicationForUnderwriterReview
  FoobaraDemo::LoanOrigination::FindALoanFileThatNeedsReview            Will return a loan file
                                                                        that needs review or nil if
                                                                        there are no loan files
                                                                        that need review
  FoobaraDemo::LoanOrigination::FindAllLoanFiles
  FoobaraDemo::LoanOrigination::GenerateLoanFilesReport
  FoobaraDemo::LoanOrigination::StartUnderwriterReview
  FoobaraDemo::LoanOrigination::StartLoanApplication
  FoobaraDemo::LoanOrigination::CreateCreditPolicy
  FoobaraDemo::LoanOrigination::FindCreditPolicy
  FoobaraDemo::LoanOrigination::ApproveLoanFile
  FoobaraDemo::LoanOrigination::AddCreditScore
  FoobaraDemo::LoanOrigination::FindLoanFile
  FoobaraDemo::LoanOrigination::DenyLoanFile
  FoobaraDemo::LoanOrigination::AddPayStub                              Add a pay stub to a loan
                                                                        file
  FoobaraDemo::LoanOrigination::Demo::PrepareDemoRecords
  FoobaraDemo::LoanOrigination::ReviewAllLoanFiles
...
$ 
```

We can see it there at the bottom. Let's see what inputs it has:


```
$ ./loan-origination --help ReviewAllLoanFiles
...
Command inputs:

 -v, --agent-options-verbose
...
 -a, --agent-name AGENT_NAME
 -l, --llm-model LLM_MODEL                             The model to use for the
  LLM. One of: chatgpt-4o-latest, claude-2.0, claude-2.1, 
  ...
  claude-3-5-sonnet-20241022, claude-3-7-sonnet-20250219, 
  ...
  claude-opus-4-20250514, claude-sonnet-4-20250514, gpt-3.5-turbo,
  ... 
  gpt-4o, gpt-4o-2024-05-13, gpt-4o-2024-08-06, gpt-4o-2024-11-20,
  ...
  Default: claude-3-7-sonnet-20250219
...
$ 
```

I've snipped many options as well as a ton of models out of the `--llm-model` option. I won't make use of the 
`--llm-model` option here but it supports ollama, openai, and anthropic models. It defaults to 
claude-3-7-sonnet-20250219.

I will make use of `--agent-options-verbose` and `--agent-name`, though. So... let's review these loan files!

```
$ ./loan-origination ReviewAllLoanFiles --agent-options-verbose --agent-name UnderwritingAgent
UnderwritingAgent: Foobara::Agent::DescribeCommand.run(command_name: "Foobara::Agent::ListCommands")
UnderwritingAgent: Foobara::Agent::ListCommands.run
UnderwritingAgent: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::FindAllLoanFiles")
UnderwritingAgent: FoobaraDemo::LoanOrigination::FindAllLoanFiles.run
UnderwritingAgent: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::StartUnderwriterReview")
UnderwritingAgent: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::FindCreditPolicy")
UnderwritingAgent: FoobaraDemo::LoanOrigination::StartUnderwriterReview.run(loan_file: 1)
UnderwritingAgent: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::FindLoanFile")
UnderwritingAgent: FoobaraDemo::LoanOrigination::FindLoanFile.run(loan_file: 1)
UnderwritingAgent: FoobaraDemo::LoanOrigination::FindCreditPolicy.run(credit_policy: 1)
UnderwritingAgent: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::ApproveLoanFile")
UnderwritingAgent: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::DenyLoanFile")
UnderwritingAgent: FoobaraDemo::LoanOrigination::DenyLoanFile.run(loan_file: 1, credit_score_used: 650, denied_reasons: ["low_credit_score"])
UnderwritingAgent: FoobaraDemo::LoanOrigination::StartUnderwriterReview.run(loan_file: 2)
UnderwritingAgent: FoobaraDemo::LoanOrigination::FindLoanFile.run(loan_file: 2)
UnderwritingAgent: FoobaraDemo::LoanOrigination::FindCreditPolicy.run(credit_policy: 2)
UnderwritingAgent: FoobaraDemo::LoanOrigination::DenyLoanFile.run(loan_file: 2, credit_score_used: 750, denied_reasons: ["insufficient_pay_stubs_provided"])
UnderwritingAgent: FoobaraDemo::LoanOrigination::StartUnderwriterReview.run(loan_file: 3)
UnderwritingAgent: FoobaraDemo::LoanOrigination::FindLoanFile.run(loan_file: 3)
UnderwritingAgent: FoobaraDemo::LoanOrigination::FindCreditPolicy.run(credit_policy: 3)
UnderwritingAgent: FoobaraDemo::LoanOrigination::ApproveLoanFile.run(loan_file: 3, credit_score_used: 750)
UnderwritingAgent: FoobaraDemo::LoanOrigination::FindAllLoanFiles.run
UnderwritingAgent: Foobara::Agent::NotifyUserThatCurrentGoalHasBeenAccomplished.run
$ 
```

So we can see all the decisions it made. Let's look at the report now:

```
$ ./loan-origination GenerateLoanFilesReport
{
  id: 1,
  applicant: "Barbara",
  state: "denied",
  underwriter_decision: {
    decision: :denied,
    credit_score_used: 650,
    denied_reasons: [
      :low_credit_score
    ]
  }
},
{
  id: 2,
  applicant: "Basil",
  state: "denied",
  underwriter_decision: {
    decision: :denied,
    credit_score_used: 750,
    denied_reasons: [
      :insufficient_pay_stubs_provided
    ]
  }
},
{
  id: 3,
  applicant: "Fumiko",
  state: "drafting_docs",
  underwriter_decision: {
    decision: :approved,
    credit_score_used: 750
  }
}
$ 
```

Looks great! It used the correct credit scores for the different loan files, 
made the correct approved/denial decisions
and chose the proper denial reason for the different denied loan files.

### A More Complex and Interesting Example

Adding an agent to a program with one line of Ruby code might be fun clickbait and all, but let's look at an
example that highlights some features of `AgentBackedCommand` that are not demonstrated in
the one-liner example above. Check this out, in a new executable called `review-all-loan-files`
we have written:

```ruby 
#!/usr/bin/env ruby

require_relative "boot"

module FoobaraDemo
  module LoanOrigination
    class UnderwriterSummary < Foobara::Model
      attributes do
        loan_file_id :integer, :required
        pay_stub_count :integer, :required
        fico_scores [:integer, :integer, :integer], :required
        credit_policy CreditPolicy, :required
      end
    end

    class ReviewLoanFile < Foobara::AgentBackedCommand
      description "Starts the underwriter review then checks requirements in its CreditPolicy " \
                  "and approves or denies accordingly."

      inputs UnderwriterSummary
      result LoanFile::UnderwriterDecision

      depends_on StartUnderwriterReview, DenyLoanFile, ApproveLoanFile

      verbose
      llm_model "claude-3-7-sonnet-20250219"
      agent_name "Inner"
    end

    class ReviewAllLoanFiles < Foobara::AgentBackedCommand
      result [{
                applicant_name: :string,
                decision: LoanFile::UnderwriterDecision
              }]

      depends_on ReviewLoanFile, FindALoanFileThatNeedsReview

      verbose
      llm_model "gpt-4o"
      agent_name "Outer"
    end
  end
end

outcome = FoobaraDemo::LoanOrigination::ReviewAllLoanFiles.run

if outcome.success?
  outcome.result.each do |name_and_decision|
    name, decision = name_and_decision.values_at(:applicant_name, :decision)

    if decision.denied?
      puts "Denied: #{name}"
      puts "Reason: #{decision.denied_reasons.join(", ")}"
    else
      puts "Approve: #{name}"
    end
    puts
  end
else
  puts outcome.errors_hash
end
```

So here we've abandoned the CLI command connector and are just using the commands directly.

We have introduced a second `AgentBackedCommand` called `ReviewLoanFile`. These two `AgentBackedCommand`s are not only using two different models but two different services entirely.

We are making use of `inputs` on `ReviewLoanFile` just like we would with any `Foobara::Command` that needs
inputs. Originally, inputs there were defined as:

```ruby
inputs do
  loan_file LoanFile, :required
end
```

But this presents a problem. Can you spot it? We want to avoid things like bias from the names of the
applicants. So we remodeled it by introducing a new model to provide only this information and called it 
`UnderwriterSummary`. 

Something cool about this
is we didn't have to change `ReviewAllLoanFiles` when we changed the inputs to `ReviewLoanFile` since 
it has no #execute method nor domain logic to update to conform to the new interface!

We are also specifying a result type for `ReviewAllLoanFiles`, like so:

```ruby
result [{
          applicant_name: :string,
          decision: LoanFile::UnderwriterDecision
        }]
```

and are making programmatic use of its outcome like so:

```ruby
if decision.denied?
  puts "Denied: #{name}"
  puts "Reason: #{decision.denied_reasons.join(", ")}"
else
  puts "Approved: #{name}"
end
```

We are guaranteed that this data
has the types we specified in `.result` just like with any other `Foobara::Command` and
can use it programmatically just fine as a result.

We are also making use of `depends_on` which is standard Foobara stuff to limit what subcommands
a command is allowed to run.

There are a few DSL methods that we're making use of here: `verbose` will let us see which decisions
which Agent is making, `llm_model` will let us specify the model for each agent independently, 
and `agent_name` will let us specify
the name for each agent independently so we can tell them apart in the output.

So let's re-create our demo records:

```
$ ./loan-origination Demo::PrepareDemoRecords
```

And now let's run this new script and review these loan files with two agents!

```
$ ./review-all-loan-files
Outer: Foobara::Agent::DescribeCommand.run(command_name: "Foobara::Agent::ListCommands")
Outer: Foobara::Agent::ListCommands.run
Outer: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::FindALoanFileThatNeedsReview")
Outer: FoobaraDemo::LoanOrigination::FindALoanFileThatNeedsReview.run
Outer: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::ReviewLoanFile")
Outer: FoobaraDemo::LoanOrigination::ReviewLoanFile.run(loan_file_id: 4, pay_stub_count: 1, fico_scores: [600, 650, 750], credit_policy: 4)
Inner: Foobara::Agent::DescribeCommand.run(command_name: "Foobara::Agent::ListCommands")
Inner: Foobara::Agent::ListCommands.run
Inner: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::StartUnderwriterReview")
Inner: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::DenyLoanFile")
Inner: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::ApproveLoanFile")
Inner: FoobaraDemo::LoanOrigination::StartUnderwriterReview.run(loan_file: 4)
Inner: FoobaraDemo::LoanOrigination::DenyLoanFile.run(loan_file: 4, credit_score_used: 650, denied_reasons: ["low_credit_score"])
Inner: Foobara::Agent::DescribeCommand.run(command_name: "Foobara::Agent::Inner::NotifyUserThatCurrentGoalHasBeenAccomplished")
Inner: Foobara::Agent::Inner::NotifyUserThatCurrentGoalHasBeenAccomplished.run(result: {"decision" => "denied", "credit_score_used" => 650, "denied_reasons" => ["low_credit_score"]})
Outer: FoobaraDemo::LoanOrigination::FindALoanFileThatNeedsReview.run
Outer: FoobaraDemo::LoanOrigination::ReviewLoanFile.run(loan_file_id: 5, pay_stub_count: 1, fico_scores: [600, 650, 750], credit_policy: 5)
Inner: Foobara::Agent::DescribeCommand.run(command_name: "Foobara::Agent::ListCommands")
Inner: Foobara::Agent::ListCommands.run
Inner: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::StartUnderwriterReview")
Inner: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::DenyLoanFile")
Inner: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::ApproveLoanFile")
Inner: FoobaraDemo::LoanOrigination::StartUnderwriterReview.run(loan_file: 5)
Inner: FoobaraDemo::LoanOrigination::DenyLoanFile.run(loan_file: 5, credit_score_used: 750, denied_reasons: ["insufficient_pay_stubs_provided"])
Inner: Foobara::Agent::DescribeCommand.run(command_name: "Foobara::Agent::Inner::NotifyUserThatCurrentGoalHasBeenAccomplished")
Inner: Foobara::Agent::Inner::NotifyUserThatCurrentGoalHasBeenAccomplished.run(result: {"decision" => "denied", "credit_score_used" => 750, "denied_reasons" => ["insufficient_pay_stubs_provided"]})
Outer: FoobaraDemo::LoanOrigination::FindALoanFileThatNeedsReview.run
Outer: FoobaraDemo::LoanOrigination::ReviewLoanFile.run(loan_file_id: 5, pay_stub_count: 1, fico_scores: [600, 650, 750], credit_policy: 5)
Inner: Foobara::Agent::DescribeCommand.run(command_name: "Foobara::Agent::ListCommands")
Inner: Foobara::Agent::ListCommands.run
Inner: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::StartUnderwriterReview")
Inner: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::DenyLoanFile")
Inner: Foobara::Agent::DescribeCommand.run(command_name: "FoobaraDemo::LoanOrigination::ApproveLoanFile")
Inner: FoobaraDemo::LoanOrigination::StartUnderwriterReview.run(loan_file: 5)
Inner: FoobaraDemo::LoanOrigination::ApproveLoanFile.run(loan_file: 5, credit_score_used: 750)
Inner: Foobara::Agent::DescribeCommand.run(command_name: "Foobara::Agent::Inner::NotifyUserThatCurrentGoalHasBeenAccomplished")
Inner: Foobara::Agent::Inner::NotifyUserThatCurrentGoalHasBeenAccomplished.run(result: {"decision" => "approved", "credit_score_used" => 750})
Outer: FoobaraDemo::LoanOrigination::FindALoanFileThatNeedsReview.run
Outer: Foobara::Agent::DescribeCommand.run(command_name: "Foobara::Agent::Outer::NotifyUserThatCurrentGoalHasBeenAccomplished")
Outer: Foobara::Agent::Outer::NotifyUserThatCurrentGoalHasBeenAccomplished.run(result: [{"applicant_name" => "Barbara", "decision" => {"decision" => "denied", "credit_score_used" => 650, "denied_reasons" => ["low_credit_score"]}}, {"applicant_name" => "Basil", "decision" => {"decision" => "denied", "credit_score_used" => 750, "denied_reasons" => ["insufficient_pay_stubs_provided"]}}, {"applicant_name" => "Fumiko", "decision" => {"decision" => "approved", "credit_score_used" => 750}}])

Denied: Barbara
Reason: low_credit_score

Denied: Basil
Reason: insufficient_pay_stubs_provided

Approved: Fumiko
```

Cool! We programmatically used result data from a command without having to write its execute method!

And if we check `./loan-origination GenerateLoanFilesReport` we can see that our program did everything correctly!

### An Interesting Takeaway...

Putting together this demo, something crossed my mind: It could be that we might find ourselves wanting to use LLMs
to handle some of this domain logic for us
as we prototype and discover the domain, but once the business scales, it might be worth the increased cost savings 
to then convert `AgentBackedCommand`s to just regular `Foobara::Command`s. Take for example `ReviewAllLoanFiles`. 
We do get some benefit of not having to manage the interface change between it and `ReviewLoanFile`, but it 
could be expensive to run if the business scales and its domain logic isn't that difficult to implement
in a typical #execute method.

This possibility is interesting to me because it is the opposite of my intuition: that we might start with a well-working domain 
and then look for opportunities to automate parts of it.
In some cases, it might be that we want the reverse: to automate various high-level domain operations during
prototyping and early stages of business and then start replacing LLM automation with
implemented domain logic as the business scales and the domain solidifies.

### Links

* The foobara-agent-backed-command gem: https://github.com/foobara/agent-backed-command
* The demo loan origination domain: https://github.com/foobara-demo/loan-origination/tree/main/src
* Code used in this demo: https://github.com/foobara/foobarticles/tree/main/src/agent_in_one_line_of_ruby_code
* Foobara: https://foobara.com

### Please Reach Out!

Want help using Foobara or want to help Foobara by contributing code/art/documentation/whatever to it? Please reach out! https://foobara.com

Disclaimers: 1) This agent just does whatever it thinks it needs to do to accomplish its goal. So
play with this with caution! 2) This loan origination domain is just to demo cool aspects of Foobara
in a relatable way. This is probably obvious, but this is not meant for making real credit decisions, which
might not even be a safe task for an LLM, anyway.

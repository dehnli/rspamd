[![CircleCI](https://circleci.com/gh/vstakhov/rspamd/tree/master.svg?style=svg)](https://circleci.com/gh/vstakhov/rspamd/tree/master)

## Introduction

[Rspamd](https://rspamd.com) is an advanced spam filtering system that allows evaluation of messages by a number of
rules including regular expressions, statistical analysis and custom services
such as URL black lists. Each message is analysed by Rspamd and given a `spam score`.

According to this spam score and the user's settings, Rspamd recommends an action for
the MTA to apply to the message, for example, to pass, reject or add a header.
Rspamd is designed to process hundreds of messages per second simultaneously, and provides a number of
useful features.

You can watch the following [introduction video](https://www.youtube.com/watch?v=_fl9i-az_Q0) from [FOSDEM-2016](http://fosdem.org) where I describe the main features of Rspamd and explain why Rspamd is so fast.

Rspamd is [packaged](https://rspamd.com/downloads.html) for the major Linux distributions, and is also available via [FreeBSD ports](https://freshports.org/mail/rspamd), NetBSD [pkgsrc](https://pkgsrc.org) and [OpenBSD ports](http://openports.se/mail/rspamd).

## Spam filtering features

The Rspamd distribution contains a number of mail processing features, including such techniques as:

* **Regular expressions filtering** - allows basic processing of messages, their textual parts, MIME headers and
SMTP data received by MTA against a set of expressions that includes both normal regular expressions and
message processing functions. Rspamd expressions are a powerful tool that allows to filter messages based on
some pre-defined rules. This feature is similar to regular expressions in the SpamAssassin spam filter.


* **SPF module** that allows to validate a message's sender against the policy defined in the DNS record of sender's domain. You can read
about SPF policies [here](http://www.openspf.org/). A number of mail systems include SPF support, such as Gmail or Yahoo Mail.


* **DKIM module** validates a message’s cryptographic signature against a public key placed in the DNS record of sender's domain. Like SPF,
this technique is widely spread and allows to validate that a message is sent from that specific domain.


* **DNS black lists** allows to estimate reputation of sender's IP address or network. Rspamd uses a number of DNS lists including such lists as
`SORBS` or `Spamhaus`. However, Rspamd doesn't trust any specific DNS list and instead uses a conjunction of estimations that allows to
avoid mistakes and false positives. Rspamd also uses positive and grey DNS lists for checking for trusted senders.


* **URL black lists** are rather similar to DNS black lists but use URLs in a message to make an estimation of the sender's reputation.
This technique is very useful for finding malicious or phished domains and filter E-mail that contains them.


* **Statistics** - Rspamd uses a Bayesian classifier based on five-grams of input. This means that the input is estimated not based on individual
words, but instead is organized in chains that are further estimated by the Bayesian classifier. This approach allows to achieve better results than
traditionally used monograms (or words literally speaking), that are described in details in the following [paper](http://osbf-lua.luaforge.net/papers/osbf-eddc.pdf).


* **Fuzzy hashes** - for checking of malicious mail patterns, Rspamd uses the so called "fuzzy hashes". Unlike normal hashes, these structures are targeted to hide
the small differences between text patterns, allowing it to find similar messages quickly. Rspamd has an internal storage of such hashes and can block mass spam sendings
quickly based on feedback from the user that specifies messages’ reputation. Moreover, this allows to feed Rspamd with data from ["honeypots"](http://en.wikipedia.org/wiki/Honeypot_(computing)#Spam_versions)
without polluting the statistical module.

Rspamd uses the conjunction of different techniques to assign the final verdict to a message. This allows to improve the overall quality of filtering, and reduce the number of
false positives (e.g. when an innocent message is badly classified as a spam one). I have tried to simplify Rspamd’s usability by adding the following elements:

* **Web interface** - Rspamd is shipped with a fully functional Ajax-based web interface, that allows one to observe Rspamd statistics, to configure rules, weights and lists, to scan
and learn messages, and to view the history of scans. The interface is self-hosted, requires zero configuration and follows the recent web applications standards. You don't need a
web server or applications server to run it - you just need to run Rspamd itself and a web browser.

* **Integration with MTAs** - Rspamd can work with the most popular mail transfer systems, such as Postfix, Exim or Sendmail. For Postfix and Sendmail, there is an [`Rmilter` project](https://github.com/vstakhov/rmilter),
whilst for Exim there are several solutions to work with Rspamd. Should you require MTA integration then please consult the [integration guide](https://rspamd.com/doc/integration.html).

* **Extensive Lua API** - Rspamd ships with hundreds of [Lua functions](https://rspamd.com/doc/lua) that enable one to write their own rules for efficient and targeted spam filtering.

* **Dynamic tables** - Rspamd allows one to specify bulk lists as "dynamic maps", that are checked at runtime for updated data. Rspamd supports file, HTTP and HTTPS maps.

## Performance

Rspamd is designed to be fast. The core of Rspamd is written in C and uses an event-driven model that allows it to process multiple messages simultaneously and without blocking.
Moreover, a set of techniques is used in Rspamd to process messages faster:

* **Finite state machines processing** - Rspamd uses specialized finite state machines for performance critical tasks to process input faster than a set of regular expressions.
Of course, it is possible to implement these machines by ordinary Perl regular expressions, but then they won't be compact or human-readable. Instead, Rspamd optimizes
such actions as headers processing, received elements extraction, and protocol operations by building the concrete automata for an assigned task.

* **Expressions optimizer** - allows Rspamd to optimize expressions by execution of `likely false` or `likely true` expressions in order in the branches. That reduces a number of
expensive expressions' calls when scanning a message.

* **Symbols optimizer** - Rspamd tries to first evaluate the rules that are frequent or inexpensive in terms of time or CPU resources. This allows it to block spam before processing of
expensive rules (rules with negative weights are always checked before other ones).

* **Event driven model** - Rspamd is designed not to block anywhere in the code, and, given that a spam check requires a lot of network operations, Rspamd can process many messages
simultaneously increasing the efficiency of shared DNS caches and other system resources. Moreover, event-driven system normally scales automatically and you won't need to do any
tuning in the most of cases.

* **Hyperscan regular expressions engine** - Rspamd utilizes the [Hyperscan](https://01.org/hyperscan) engine to match multiple regular expressions at the same time. You can read the following [presentation](https://highsecure.ru/rspamd-hyperscan.pdf) where the main benefits of Hyperscan are described.

* **Clever choice of data structures** - Rspamd tries to use the optimal data structure for each task. For example, it uses very efficient suffix tries for fast matching of text
against a set of multiple patterns. Or it uses a radix bit trie for storing IP addresses information that provides O(1) access time complexity.

## Extensions

Besides its C core, Rspamd provides an extensive [Lua](http://lua.org) API to access almost all the features available directly from C. Lua is an extremely easy
to learn programming language though it is powerful enough to implement complex mail filters. In fact Rspamd has a significant amount of code written completely in Lua such as
DNS blacklists checks, user's settings or different maps implementation. You can also write your own filters and rules in Lua adopting Rspamd functionality to your needs.
Furthermore, Lua programs are very fast and their performance is rather [close](http://attractivechaos.github.io/plb/) to pure C. However, you should mention that for the most
of performance critical tasks you usually use the Rspamd core functionality than Lua code. Anyway, you can also use LuaJIT with Rspamd if your goal is maximum performance.
From the Lua API you can do the following tasks:

* **Reading the configuration parameters** - Lua code has the full access to the parsed configuration knobs and you can easily modify your plugins behaviour by means of the main
Rspamd configuration.

* **Registering custom filters** - it is more than simple to add your own filters to Rspamd: just add new index to the global variable `rspamd_config`:

~~~lua
rspamd_config.MYFILTER = function(task)
-- Do something
end
~~~

* **Full access to the content of messages** - you can access text parts, headers, SMTP data and so on and so forth by using of `task` object. The full list of methods could be found
[here](https://rspamd.com/doc/lua/task.html).


* **Pre- and post- filters** - you can register callbacks that are called before or after messages processing to make results more precise or to make some early decision,
for example, to implement a rate limit.

* **Registering functions for Rspamd** - you can write your own functions in Lua to extend Rspamd internal expression functions.

* **Managing statistics** - Lua scripts can define a set of statistical files to be scanned or learned for a specific message allowing to create more complex
statistical systems, e.g. based on an input language. Moreover, you can even learn Rspamd statistic from Lua scripts.

* **Standalone Lua applications** - you can even write your own worker based on Rspamd core and performing some asynchronous logic in Lua. Of course, you can use the
all features from Rspamd core, including such features as non-blocking IO, HTTP client and server, non-blocking Redis client, asynchronous DNS, UCL configuration and so on
and so forth.

* **API documentation** - Rspamd Lua API has an [extensive documentation](https://rspamd.com/doc/lua) where you can find examples, references and the guide about how to extend
Rspamd with Lua.


## References

* Home site: <https://rspamd.com>
* Development: <https://github.com/vstakhov/rspamd>

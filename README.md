# rfc5322 Headers Parser (typescript)

Parse [RFC5322](https://datatracker.ietf.org/doc/html/rfc5322#section-4.1) email headers into typed objects.

This project is new and currently in WIP. It is currently not tested very well. The strategy is to literally just take the specification, word for word more or less, and convert it into a PEG grammar. There are many tools that can take such a grammar and produce an AST. For this project I'm using [tsPEG](https://github.com/EoinDavey/tsPEG), which produces a parser that can be used in typescript.

## Help Wanted

I think the most useful help I could get at this point is writing more tests.

## The Grammar

Here is a copy of the grammar (thus far)

```peg
// RFC 5322
// See: https://datatracker.ietf.org/doc/html/rfc5322
// The following is the RFC5322 "Internet Message Format" specification
// using tspeg to represent the grammar and to generate a parser in TS

// Note: this first token is the entrypoint for the rest of the grammar
input           := addr_spec

// "Core Rules" from RFC5234
// See: https://datatracker.ietf.org/doc/html/rfc5234#appendix-B.1
CR              :=    '\x0D'        // carriage return, i.e. '\r'
CRLF            :=    CR LF         // Internet standard newline
DQUOTE          :=    '\x22'        // double quote, i.e. '"'
HTAB            :=    '\x09'        // horizontal tab, i.e. 'TAB'
LF              :=    '\x0A'        // linefeed, i.e. '\n'
SP              :=    '\x20'        // space, i.e. 'Space'
VCHAR           :=    '[\x21-\x7E]' // visible (printing) characters
WSP             :=    SP | HTAB     // white space

// § 3.2.1 Quoted Characters
// See: https://datatracker.ietf.org/doc/html/rfc5322#section-3.2.1
quoted_pair     :=    {'\\' {VCHAR | WSP}} | obs_qp

// §3.2.2 Folding White Space and Comments
// See: https://datatracker.ietf.org/doc/html/rfc5322#section-3.2.2
FWS             :=    {{WSP* CRLF}? WSP+} |  obs_FWS
ctext           :=    '[\x21-\x27]' | '[\x2a-\x5b]' | '[\x5d-\x7e]' | obs-ctext
ccontent        :=    ctext | quoted_pair | comment
comment         :=    '\(' {FWS? ccontent}* FWS? '\)'
CFWS            :=    {{FWS? comment}+ FWS?} | FWS

// § 3.2.3 Atom
// See: https://datatracker.ietf.org/doc/html/rfc5322#section-3.2.3
// (Printable US-ASCII characters not including specials. Used for atoms.)
atext           :=    atext='[A-Za-z0-9!#$%&\x27*+\-\/=?^_`{|}~]'
atom            :=    CFWS? atext+ CFWS?
dot_atom_text   :=    head_atext = atext+ {'\.' tail_atext = atext+}*
dot_atom        :=    CFWS? dot_atom_text = dot_atom_text CFWS?
// (Special characters that do not appear in atext)
specials        :=     '\(' |'\)' | '[<>]' | '\[' | '\]' | '[:;@]' | '\\' | ',' | '\.' | DQUOTE

// § 3.2.4 Quoted Strings
// See https://datatracker.ietf.org/doc/html/rfc5322#section-3.2.4
// (Printable US-ASCII characters not including '\"' or the quote character)
qtext           :=    '\x21' | '[\x23-\x5b]' | '[\x5d-\x7e]' | obs_qtext
qcontent        :=    qtext | quoted_pair
quoted_string   :=    CFWS? DQUOTE {FWS? qcontent = qcontent}* FWS? DQUOTE CFWS?

// § 3.2.5 Miscellaneous Tokens
// See: https://datatracker.ietf.org/doc/html/rfc5322#section-3.2.5
word            :=    atom | quoted_string

// § 3.4.1 Addr-Spec Specification
// See: https://datatracker.ietf.org/doc/html/rfc5322#section-3.4.1
addr_spec       :=    local_part = local_part '@' domain = domain
local_part      :=    dot_atom = dot_atom | quoted_string = quoted_string | obs_local_part = obs_local_part
domain          :=    dot_atom = dot_atom | domain_literal = domain_literal | obs_domain = obs_domain
domain_literal  :=    CFWS? '\[' {FWS? dtext=dtext}* FWS? '\]' CFWS?
// (Printable US-ASCII characters not including '[', ']', or '\"')
dtext           :=    '[\x21-\x5a]' | '[\x5e-\x7e]' | obs_dtext = obs_dtext

// § 4.1 Miscellaneous Obsolete Tokens
// See: https://datatracker.ietf.org/doc/html/rfc5322#section-4.1
// (US-ASCII control characters that do not include the carriage return, line feed, and white space characters)
obs_NO_WS_CTL   :=    '[\x01-\x08]' | '\x0B' | '\x0C' | '[\x0E-\x1F]' | '\x7F'
obs_qtext       :=    obs_NO_WS_CTL
obs_ctext       :=    obs_NO_WS_CTL
obs_qp          :=    '\\' {'\x00' | obs_NO_WS_CTL | LF | CR}

// § 4.2 Obsolete Folding White Space
// See: https://datatracker.ietf.org/doc/html/rfc5322#section-4.2
obs_FWS         :=    WSP+ {CRLF WSP+}*

// § 4.4 Obsolete Addressing
// See: https://datatracker.ietf.org/doc/html/rfc5322#section-4.4
obs_local_part  :=    word {'\.' word}*
obs_domain      :=    head_atom = atom {'\.' tail_atom = atom}*
obs_dtext       :=    obs_NO_WS_CTL | quoted_pair
```
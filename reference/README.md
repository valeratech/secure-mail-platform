# Reference configuration

The clean, authoritative configuration for each component, populated **only** by copying
it off the running mail VM after a section is accepted — never by typing configuration into
the repo. It claims to be what is actually running; hand-authored config stops being true
within days.

Subdirectories are created as their sections land (`bind/`, `postfix/`, `dovecot/`,
`rspamd/`, `roundcube/`, `nftables/`, …). Every non-obvious directive is annotated with what
it mitigates and the test that would fail if it were removed.

Private keys, CA state, DKIM keys, and credentials never appear here.

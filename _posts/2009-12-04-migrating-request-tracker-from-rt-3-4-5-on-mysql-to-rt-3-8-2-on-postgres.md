---
layout: post
title: "Migrating request-tracker (from rt-3.4.5 on mysql to rt-3.8.2 on postgres)"
date: 2009-12-04
categories: db mysql perl postgresql request-tracker rt solnet
---

I've recently had the "pleasure" of upgrading our company's request tracker system and migrating it from MySQL to Postgres and I'd like to share what I found out.

The process of installing and configuring the request-tracker-3.8 package from Debian experimental is considered obvious (and user-specific) and won't be covered here.

First of all, all the data had to be migrated, except for the 'sessions' table which should be created empty (what would we need old sessions for).

<code>
export LC_ALL=C
TABLES=ACL Attachments Attributes CachedGroupMembers CustomFieldValues CustomFields GroupMembers Groups Links ObjectCustomFieldValues ObjectCustomFields Principals Queues ScripActions ScripConditions Scrips Templates Tickets Transactions Users
for foo in $TABLES; do echo $foo; mysqldump rtdb "$foo" --default-character-set=utf8 --compatible=postgresql --compact -t -u rtuser --password=omitted -r rtdb-"$foo"-`date +%s`.sql; done
</code> 

I chose to export each table into a different file to ease handling and because vim takes too long on 400MB files.

Mysqldump may have the --compatible=postgresql option, but don't expect the output will be postgresql compatible .. but maybe it used to at some point, nowadays, Postgres won't accept string fields <tt>'like \' \\this'</tt>, but only <tt>'like '' \this'</tt> or you have to explicitly tell it to see escaping .. <tt>E'like \' \\this'</tt>.

Also, postgres won't handle the quotes in INSERT INTO "Table" .. .

<code>
perl -i -npe 's/^INSERT INTO "(\w+)"/INSERT INTO \L$1/; '"s/,'/,E'/g;" rtdb-*.sql
</code>

Worked just fine.

Seeing the inserts, I really didn't want to handle transmogrifying the CREATE TABLE statements and such, so I exported a freshly created pg-8.3 database from rt-3.8.2, removed any data from the dump and split it into the HEAD (the part before data) and TAILS (the part after). 

The only schema difference that had to be handled was in the table CustomFields, where order of columns was changed.

<code>
perl -i -npe 's/INSERT INTO customfields VALUES/INSERT INTO customfields (id,name,type,maxvalues,pattern,repeated,description,sortorder,lookuptype,creator,created,lastupdatedby,lastupdated,disabled) VALUES/' rtdb-CustomFields-*.sql
</code>

Explicit insert helped :).

This made the data ready to be loaded to a new, empty database. Almost.

For some reason RT uses text fields for saving mail content so non-utf8 mails couldn't be imported into the UTF-8 database .. after struggling long and hard, I didn't actually find a proper way to convert it so I used a quick and dirty hack and simply stripped all diacritics from the input data .. the language was Czech and cp1250, iso-8859-2 and utf-8 encodings were used .. I wrote a simple diacritics stripper for Czech that doesn't need to know the source encoding (because various were mixed in a single table dump) .. so that's what the zz_subs filter does in the following command.

<code>
psql rtdb &lt; ok/HEADS
for foo in rtdb-*.sql; do echo $foo; cat "$foo" | ../zz_subs | psql rtdb; done
psql rtdb &lt; ok/TAILS
</code>

Thus the data was migrated but RT still kept falling when I tried to insert anything new .. the culprit were the postgresql sequences (used to generate ids) - they didn't get updated because all entried were inserted with explicit id .. so all sequences still tried to use 1 as a new id, which didn't work. (rant mode: is it just me or is this behaviour incredibly stupid? should they at least keep incrementing if the next value is already there?).

<code>
psql rtdb &lt;&lt;EOF
select setval('acl_id_seq', (select max(id) from acl));
select setval('attachments_id_seq', (select max(id) from attachments));
select setval('attributes_id_seq', (select max(id) from attributes));
select setval('cachedgroupmembers_id_seq', (select max(id) from cachedgroupmembers));
select setval('customfields_id_seq', (select max(id) from customfields));
select setval('customfieldvalues_id_seq', (select max(id) from customfieldvalues));
select setval('groupmembers_id_seq', (select max(id) from groupmembers));
select setval('groups_id_seq', (select max(id) from groups));
select setval('links_id_seq', (select max(id) from links));
select setval('objectcustomfields_id_s', (select max(id) from objectcustomfields));
select setval('objectcustomfieldvalues_id_s', (select max(id) from objectcustomfieldvalues));
select setval('principals_id_seq', (select max(id) from principals));
select setval('queues_id_seq', (select max(id) from queues));
select setval('scripactions_id_seq', (select max(id) from scripactions));
select setval('scripconditions_id_seq', (select max(id) from scripconditions));
select setval('scrips_id_seq', (select max(id) from scrips));
select setval('templates_id_seq', (select max(id) from templates));
select setval('tickets_id_seq', (select max(id) from tickets));
select setval('transactions_id_seq', (select max(id) from transactions));
select setval('users_id_seq', (select max(id) from users));
EOF
</code>

Deep magic, but works.

Now, rt has it's own thingy for upgrading database for rt upgrades, so I wanted to run it so everything is in order. After skimming through, it does seem to do some useful stuff. Unfortunately there's a schema change in 3.7.3 which tried to remove all tables so 3.7.3 has to be skipped.

<code>
(echo 3.4.5; echo 3.7.1; echo y) | rt-setup-database --action upgrade --dba-password=omitted
(echo 3.7.3; echo; echo y) | rt-setup-database --action upgrade --dba-password=omitted
</code>

And really, that's all you need to do to migrate a rt.

You might also want to put something like this in your crontab:
<code>
0 4 * * *   postgres    /bin/echo "delete from sessions where LastUpdated < (now() - '24 hour'::interval);" | /usr/bin/psql rtdb 
</code>

Rant mode on:
Except, it didn't really work. And why not, you ask? Because of ****** cpan putting his sleazy libraries where they don't belong. Why, oh why can't you set cpan to install to /usr/local or somewhere that has less priority than /usr/share/ where debian puts them. It didn't work because we had Net::LDAP from 1999 in /usr/lib (and a current version in /usr/share that didn't get used).
Oh well :)

---

Comments:

    (December 8, 2010 at 9:09 PM)
    
    Thanks for this blog posting, it saved me a lot of time when doing my own migration from MySQL to Postgres. 
    
    One hint regarding the character encoding: Using a Perl script, using an explicit conversion to UTF-8 using the encode() function works wonders. Pseudocode:
    
    MySQL: (SELECT * FROM Attachments)
    Prepare PgSQL: (INSERT INTO Attachments (?, ?, ....)
    
    For all rows in SELECT{
    push(@bind_values, $row[...]); 
    ...
    push(@bind_values, encode("UTF-8", $row[...]);
    PreparedStatement->execute(@bind_values);
    }
    
    This is pretty slow, but OK for a one-time task.
    
    -- Matthias

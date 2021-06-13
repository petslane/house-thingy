---
layout: post
title: Optimizing HomeAssistant database storage
excerpt_separator: <!--more-->
---

This post is about HomeAssistant and how to reduce database size when using [Recorder](https://www.home-assistant.io/integrations/recorder/) integration to log sensor data for a long period of time. By default `recorder` integration saves all state changes in the `states` table and all events in the `events` table. It keeps only the last 10 days worth of data, auto-clearing older data. For the DB engine, SQLite is used by default (in this post, Postgres database engine will be used instead).

<!--more-->

In my case, this SQLite database size is 45MB. This does not sound too much, but this is only for 10 days. If I would want to have data for a longer period - 1 year, then the database file would be 1.6GB. And depending on how many sensors you have and how often they report a status change, the amount of data can be a lot bigger. And of course, you would like to store data for longer than 1 year.

One way to reduce the amount of data is to exclude some sensors that will not be logged in DB, or select only specific sensors for logging. I personally do not want to exclude anything as I do not know if I need it in the future or not. Better to log as much as possible as long as possible.

Looking at the database, the main tables are `events` and `states`. Every time something happens in HA, it will be logged into the `events` table. For me it does not contain anything interesting to keep it for a long time. `states` table contains sensor state changes, but it contains a lot of data I do not need. I am pretty much interested only in sensor name (entity), state, and date-time of state change. Fields that I do no need are `domain`, `attributes`, `event_id`, `last_changed`, `last_updated` and `old_state_id`. Also, the entity name is `varchar(255)` field in `states` table, but these values repeat a lot and take a lot of space. Better to store entity names in a different table and reference them from `states` table by primary key.

So, here is my example setup on how to store states more efficiently for a longer period. I used Postgres DB engine for storing data, so configure HA recorder to use Postgres. You can keep default 10 day auto-clearing. We are going to create separate tables for states and entities, and fill them automatically with triggers whenever HA adds a state to the default `states` table.

So, first, let's create a `dwh_entity` table, this will store all entities:
```
CREATE TABLE dwh_entity
(
	id SERIAL NOT NULL CONSTRAINT dwh_entity_pk PRIMARY KEY,
	entity VARCHAR(255) NOT NULL
);

CREATE UNIQUE INDEX dwh_entity_entity_uindex ON dwh_entity (entity);
```

And then `dwh_data` table that stores all state changes:
```
CREATE TABLE dwh_data
(
	id INT NOT NULL CONSTRAINT dwh_data_pk PRIMARY KEY,
	entity_id INT NOT NULL CONSTRAINT dwh_data_dwh_entity_id_fk REFERENCES dwh_entity ON UPDATE CASCADE ON DELETE CASCADE,
	state VARCHAR(255) NOT NULL,
	created TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE INDEX dwh_data_created_entity_id_index ON dwh_data (created, entity_id);
```

And to make it easier to select data (and also add data), let's make a view that joins these two tables:
```
CREATE OR REPLACE VIEW data_v AS SELECT d.id, e.entity, d.state, d.created FROM dwh_data d JOIN dwh_entity e on d.entity_id = e.id;
```

Now it's time to do some Postgres magic. Usually, if the view selects from one table, then this view can be used to insert data, but if it selects from multiple tables, then inserts do not work. Unless you create `INSTEAD OF INSERT` trigger, that does inserting whenever something tries to insert into view. So here we go:
```
CREATE OR REPLACE FUNCTION InsertIntoDataView() RETURNS trigger AS $$
DECLARE
    e_id integer;
BEGIN
    SELECT id INTO e_id FROM dwh_entity WHERE entity = NEW.entity;
    IF e_id IS NULL THEN
        INSERT INTO dwh_entity (entity)
		     VALUES (NEW.entity)
		ON CONFLICT (entity) DO NOTHING
		  RETURNING id INTO e_id;
    END IF;
    INSERT INTO dwh_data (id, entity_id, state, created) VALUES (NEW.id, e_id, NEW.state, NEW.created) ON CONFLICT DO NOTHING;
    RETURN NEW;
END; $$ LANGUAGE PLPGSQL;

CREATE TRIGGER data_v_on_insert INSTEAD OF INSERT ON data_v FOR EACH ROW EXECUTE PROCEDURE InsertIntoDataView();
```

We created `InsertIntoDataView` function that takes in inserted data, finds existing entity id from `dwh_entity` table, if it does not exist, then inserts it and uses the id of entity to store data into `dwh_data` table. And then create trigger that executes that function whenever someone inserts data into view.

Now we need another trigger that whenever HA adds new data to the existing `states` table, then same state change will be inserted into our `data_v` view:
```
CREATE FUNCTION InsertIntoStates() RETURNS TRIGGER
	LANGUAGE PLPGSQL
AS $$
BEGIN
   INSERT INTO data_v (id, entity, state, created)
   VALUES (NEW.state_id, NEW.entity_id, NEW.state, NEW.created);

   RETURN NEW;
END;
$$;

 CREATE TRIGGER states_on_insert
  AFTER INSERT ON states FOR EACH ROW
EXECUTE PROCEDURE InsertIntoStates();
```

Here, the new function `InsertIntoStates` will be executed after something is added to the `states` table it also inserts the same entity name and state into the `data_v` view.

If you already have data in the `states` table before the last trigger was created and you what to have them in the `data_v` view, then you can just insert them from the `states` table:
```
INSERT INTO data_v (id, entity, state, created) SELECT s.state_id, s.entity_id, s.state, s.created FROM states s ON CONFLICT DO NOTHING;
```

Now you can point Grafana or other tools to the `data_v` view for selecting data.

In my case, for the same time period, the `events`+`states` table size compared to the `dwh_data`+`dwh_entity` table size is reduced 6 times. Or if you compare the `states` table with `dwh_data`+`dwh_entity` tables, then it still more than 3 times smaller.

With this setup, I can now just keep logging everything and not care about database size as instead of 1.6GB per year, it grows about 266MB per year.
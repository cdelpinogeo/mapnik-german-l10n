#  OSM map l10n functions
## (from german mapnik style)

All l10n functions from german mapnik style are implemented as PL/pgSQL stored procedures
and are therefore usable in a renderer independent way.

For this reason they are now hosted in their own repository.

Currently the code consists of three parts:

1. An "Any-Latin" transliterate function using libicu
2. A japanese kanji transliterate function using libkakasi
3. A couple of PL/pgSQL functions which can be used to generate labels for
   map rendering.

See **INSTALL.md** file from sources for manual installation instructions.
If you just installed the debian package all you have to do now ist to enable
our extension in your PostgreSQL database as follows:

```sql
CREATE EXTENSION osml10n;
```

### API
The following functions are provided for use in map rendering:

__`osml10n_get_placename(name text, local_name text, int_name text, name_en text, loc_in_brackets boolean, show_brackets boolean DEFAULT false, separator text DEFAULT chr(10), place geometry DEFAULT NULL)`__
:	Will try its best to return a usable name pair with name in brackets (or vise versa if loc_in_brackets is set)

__`osml10n_get_streetname(name text, local_name text, int_name text, name_en text, loc_in_brackets boolean, how_brackets boolean DEFAULT false, separator text DEFAULT ' - ', langcode text DEFAULT 'de', place geometry DEFAULT NULL)`__
:	Same as get_localized_placename, but with some common abbreviations for street names (Straße->Str.), if name ist longer than 15 characters

__`osml10n_get_name_without_brackets(name text, local_name text, int_name text, name_en text, place geometry DEFAULT NULL)`__
:	Same as get_localized_placename, but with no names in brackets

__`osml10n_get_placename_from_tags(tags hstore, loc_in_brackets boolean, show_brackets boolean DEFAULT false, separator text DEFAULT chr(10), targetlang text DEFAULT 'de', place geometry DEFAULT NULL)`__
:	Same as osml10n_get_placename but with hstore column input


__`osml10n_get_streetname_from_tags(tags hstore, loc_in_brackets boolean, show_brackets boolean DEFAULT false, separator text DEFAULT ' - ', targetlang text DEFAULT 'de', place geometry DEFAULT NULL)`__
:	Same as osml10n_get_streetname but with hstore column input

__`osml10n_get_name_without_brackets_from_tags(tags hstore, loc_in_brackets boolean, targetlang text DEFAULT 'de', place geometry DEFAULT NULL)`__
:	Same as osml10n_get_placename but with hstore column input

__`osml10n_get_country_name(tags hstore, separator text DEFAULT chr(10), targetlang text DEFAULT 'de')`__
:	Generate a combined country name from name:xx tags (targetlang plus official languages of the country)


A convinient way of using these functions is to hide them behind virtual colums using database views.

### Examples

#### Old style
```
select osml10n_get_placename('Москва́','Moskau',NULL,'Moscow',true) as name;
      -->	Москва́
		Moskau
select osml10n_get_placename('Москва́','Moskau',NULL,'Moscow',false) as name;
      -->	Moskau
		Москва́
select osml10n_get_placename('القاهرة','Kairo','Cairo','Cairo',false) as name;
      -->	Kairo
		القاهرة
select osml10n_get_placename('Brixen Bressanone','Brixen',NULL,NULL,false) as name;
      -->	Brixen Bressanone
select osml10n_get_placename('Roma','Rom',NULL,NULL,false) as name;
      -->	Rom
		Roma
select osml10n_get_streetname('Doktor-No-Straße',NULL,NULL,NULL,false) as name;
      -->	Dr.-No-Str.
select osml10n_get_streetname('Dr. No Street','Professor-Doktor-No-Straße',NULL,NULL,false) as name;
      -->	Prof.-Dr.-No-Str. - Dr. No St.
select osml10n_get_name_without_brackets('Dr. No Street','Doktor-No-Straße',NULL,NULL) as name;
      -->	Doktor-No-Straße
select osml10n_get_streetname('улица Воздвиженка',NULL,NULL,'Vozdvizhenka Street',true,true,' ','de') as name;
      -->	ул. Воздвиженка (Vozdvizhenka St.)
select osml10n_get_streetname('улица Воздвиженка',NULL,NULL,NULL,true,true,' ','de') as name;
      -->	ул. Воздвиженка (ul. Vozdviženka)
select osml10n_get_streetname('вулиця Молока',NULL,NULL,NULL,true,false,' - ','de') as name;
      -->	вул. Молока - vul. Moloka
```

#### Using hstore column containing all name tags from Openstreetmap
```
select osml10n_get_placename_from_tags('"name"=>"Москва́","name:de"=>"Moskau","name:en"=>"Moscow"',true) as name;
       -->	Москва́
		Moskau
select osml10n_get_placename_from_tags('"name"=>"Москва́","name:de"=>"Moskau","name:en"=>"Moscow"',false) as name;
       -->	Moskau
		Москва́
select osml10n_get_placename_from_tags('"name"=>"القاهرة","name:de"=>"Kairo","int_name"=>"Cairo","name:en"=>"Cairo"',false) as name;
       -->	Kairo
		القاهرة
select osml10n_get_placename_from_tags('"name"=>"Brixen Bressanone","name:de"=>"Brixen"',false) as name;
       -->	Brixen Bressanone
select osml10n_get_placename_from_tags('"name"=>"Roma","name:de"=>"Rom"',false) as name;
       -->	Rom
		Roma
select osml10n_get_streetname_from_tags('"name"=>"Doktor-No-Straße"',false) as name;
       -->	Dr.-No-Str.
select osml10n_get_streetname_from_tags('"name"=>"Dr. No Street","name:de"=>"Professor-Doktor-No-Straße"',false) as name;
       -->	Prof.-Dr.-No-Str. - Dr. No St.
select osml10n_get_name_without_brackets_from_tags('"name"=>"Dr. No Street","name:de"=>"Doktor-No-Straße"') as name;
       -->	Doktor-No-Straße
select osml10n_get_streetname_from_tags('"name"=>"улица Воздвиженка","name:en"=>"Vozdvizhenka Street"',true,true,' ','de') as name;
       -->	ул. Воздвиженка (Vozdvizhenka St.)
select osml10n_get_streetname_from_tags('"name"=>"улица Воздвиженка"',true,true,' ','de') as name;
       -->	ул. Воздвиженка (ul. Vozdviženka)
select osml10n_get_streetname_from_tags('"name"=>"вулиця Молока"',true,false,' - ','de') as name;
       -->	вул. Молока - vul. Moloka
select osml10n_get_placename_from_tags('"name"=>"주촌  Juchon", "name:ko"=>"주촌","name:ko_rm"=>"Juchon"',false) as name;
       -->	주촌  Juchon
select osml10n_get_placename_from_tags('"name"=>"주촌", "name:ko"=>"주촌","name:ko_rm"=>"Juchon"',false) as name;
       -->	Juchon
		J주촌
select osml10n_get_country_name('"ISO3166-1:alpha2"=>"IN","name:de"=>"Indien","name:hi"=>"भारत","name:en"=>"India"') as name;
       -->	Indien
		भारत
		India
```

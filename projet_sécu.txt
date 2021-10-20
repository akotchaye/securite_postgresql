create database dbaudit;
/*drop database Audit;*/
/*dur*/
create table ttt_company(
id_company serial primary key,
company_name varchar(200) not null,
isdelete boolean
);
/*dur*/
create table ttt_project(
id_project serial primary key,
id_company integer not null,
project_name varchar(200) not null,
isdelete boolean,
foreign key (id_company) references ttt_company(id_company)
);

/*duir*/
create table ttt_task ( 
id_task serial primary key ,
id_project integer not null,
task_name varchar(200) not null,
isdelete boolean,
foreign key (id_project) references ttt_project(id_project)
);

/*u*/
create table  ttt_user(
username varchar(10) unique primary key,
first_name varchar(100) not null,
last_name varchar(100) not null,
email varchar(100) not null,
pasword varchar(100) not null,
admin_role char(4) not null,
isdelete boolean
);

create table ttt_task_log(
id_task_log serial primary key ,
id_task integer not null,
username varchar(10) not null ,
task_description varchar(2000)not null,
task_log_date date not null,
task_isprocessing boolean not null,
task_end_date date not null,
task_minutes integer not null,
foreign key(id_task) references ttt_task(id_task),
foreign key(username) references ttt_user(username)
);

create table ttt_company_track(
id_company_t serial not null,
id_company integer not null,
company_name_t varchar(200) not null,
foreign key(id_company) references ttt_company(id_company)
);

create table ttt_project_track(
id_project_t serial not null,
id_project integer not null,
id_company_t integer not null,
project_name_t varchar(200) not null,
foreign key(id_project) references ttt_project(id_project)
);

create table ttt_task_track( 
id_task_t serial not null ,
id_task integer not null ,
id_project_t integer not null,
task_name_t varchar(200) not null,
foreign key(id_task) references ttt_task(id_task)
);

create table  ttt_user_track(
id_user_t serial not null,
username varchar(10) not null,
first_name_t varchar(100) not null,
last_name_t varchar(100) not null,
email_t varchar(100) not null,
pasword_t varchar(100) not null,
admin_role_t char(4) not null,
foreign key(username) references ttt_user(username)
);

create table ttt_task_log(
id_task_log serial primary key ,
id_task integer not null,
username varchar(10) not null ,
task_description varchar(2000)not null,
task_log_date date not null,
task_isprocessing boolean not null,
task_end_date date not null,
task_minutes integer not null,
foreign key(id_task) references ttt_task(id_task),
foreign key(username) references ttt_user(username)
);

create procedure delete_task(idt integer)
language plpgsql
as$$
begin  
  update ttt_task /*revoir les contraintes des tables pour la mise a jour*/
  set isdelete = true
  where id_task = idt ;
commit;
end;$$ 
/*procedure de suppression d'un projet*/
create procedure delete_project(idp integer)
 language plpgsql
 as$$
declare 
ligne ttt_task%rowtype ;
begin 
 update ttt_project
 set isdelete = true
 where id_project = idp ;

 for ligne in select * from ttt_task where id_project=idp
    loop
     update ttt_task 
     set isdelete = true
    end loop;

  commit;
end;$$ 
/*procedure de suppression d'une compagnie*/
create procedure delete_company(idc integer)
 language plpgsql
 as$$
declare /*déclaration de variables*/
lignep ttt_project%rowtype;
lignet ttt_task%rowtype;

begin 
/*suppression de la compagnie*/
 update ttt_company
 set isdelete = true
 where id_company = idc;
/*suppression des projets associés*/
 for lignep in select * from ttt_project where id_company=idc
    loop
     update ttt_project 
     set isdelete = true
    end loop;
/*suppression des taches associées*/
 for lignet in select * from ttt_task where id_project in 
 select id_project from ttt_project where id_company=idc
    loop
     update ttt_task 
     set isdelete = true
    end loop;

  commit;
end;$$ 

/***/

/*fonction de suppression du trigger de tt_task*/
create function  delete_function()
returns trigger
as $$
declare
tdescription ttt_task_log.task_description%TYPE;
id ttt_task.id_task%TYPE := NEW.id_task;
log_date timestamp := now();
process boolean := false;

begin
  tdescription := 'suppression de la tache  id_task : %  id_project : %  task_name: %  by : %  at : %',NEW.id_task,NEW.id_project,NEW.task_name,current_user,log_date;
  insert into ttt_task_log
  values(id,current_user,tdescription,log_date::date,process,Now()::date,extract(epoch from(now()-log_date))/60);

return null;
end; 
$$ language plpgsql;

/*trigger de suppression dans ttt_task*/
create trigger delete_trigger 
after update of isdelete on ttt_task,
for each row 
execute procedure delete_function();
/****/
/*fonction de mise a jour du trigger de company*/
create function update_company_function()
returns trigger
as$$
begin
insert into ttt_company_track 
values (OLD.id_company,OLD.company_name)
return null;
end;
$$ language plpgsql;
/*trigger de mise a jour des données*/
create trigger update_trigger
before update on ttt_company
for each row 
execute procedure update_company_function();

/*fonction de mise a jour du trigger de project*/
create function update_project_function()
returns trigger
as$$
begin
insert into ttt_project_track 
values (OLD.id_project,OLD.id_company,OLD.project_name)
return null;
end;
$$ language plpgsql;
/*trigger de mise a jour des données*/
create trigger update_project_trigger
before update on ttt_project
for each row 
execute procedure update_project_function();

/*fonction de mise a jour du trigger de task*/
create function update_task_function()
returns trigger
as$$
begin
insert into ttt_task_track 
values (OLD.id_task,OLD.id_project,OLD.task_name)
return null;
end;
$$ language plpgsql;
/*trigger de mise a jour des données*/
create trigger update_task_trigger
before update on ttt_task
for each row 
execute procedure update_task_function();

/*fonction de mise a jour du trigger de user*/
create function update_user_function()
returns trigger
as$$
begin
insert into ttt_user_track 
values (OLD.username,OLD.first_name,OLD.last_name,OLD.email,OLD.pasword,OLD.admin_role)
return null;
end;
$$ language plpgsql;
/*trigger de mise a jour des données*/
create trigger update_user_trigger
before update on ttt_user
for each row 
execute procedure update_user_function();

/*fonction d'insertion du trigger de ttt_task*/
create function  insert_task_function()
returns trigger
as $$
declare
tdescription ttt_task_log.task_description%TYPE;
log_date timestamp := now();
process boolean := false;

begin
  tdescription := 'insertion de la tache  id_task : %  id_project : %  task_name: %  by : %  at : %',id_task,id_project,task_name,current_user,log_date;
  insert into ttt_task_log
  values(id_task,current_user,tdescription,log_date::date,process,Now()::date,extract(epoch from(now()-log_date))/60);

return null;
end; 
$$ language plpgsql;

/*trigger d'insertion  dans ttt_task*/
create trigger insert_task_trigger 
after insert on ttt_task,
for each row 
execute procedure insert_task_function();

/*procedure de selection du projet*/
create procedure select_project(idp integer)
 language plpgsql
 as$$
declare 
result ttt_project%rowtype;
begin 
  for result in select * from ttt_project where id_project=idp
    loop
     raise notice '% % %',result.id_project,result.project_name,result.id_company;
    end loop;
  commit;
end;$$ 

/*procedure de selection de la compagnie*/
create procedure select_company(idc integer)
 language plpgsql
 as$$
declare 
result ttt_company%rowtype;
begin 
  for result in select * from ttt_company where id_company=idc
    loop
     raise notice '% %',result.id_company,result.company_name;
    end loop;
  commit;
end;$$ 

/*procedure de selection du task*/
create procedure select_task(idt integer)
 language plpgsql
 as$$
declare 
result ttt_task%rowtype;
begin 
  for result in select * from ttt_task where id_task=idt
    loop
     raise notice '% % %',result.id_task,result.id_project,result.task_name;
    end loop;
  commit;
end;$$ 



/********test********/
create table table1(
id1 serial primary key,
nom1 varchar (20) not null);

create table table2(
id2 serial primary key,
id1 integer not null,
nom2 varchar (20) not null,
foreign key (id1) references table1(id1) );


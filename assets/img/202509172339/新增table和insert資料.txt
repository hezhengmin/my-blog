-- create tables

create table departments (
    id          number default on null to_number(sys_guid(), 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX') 
                constraint departments_id_pk primary key,
    name        varchar2(255 char) not null,
    location    varchar2(4000 char),
    country     varchar2(4000 char)
);


create table employees (
    id               number default on null to_number(sys_guid(), 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX') 
                     constraint employees_id_pk primary key,
    department_id    number                     constraint employees_department_id_fk
                     references departments,
    name             varchar2(50 char) not null,
    email            varchar2(255 char),
    cost_center      number,
    date_hired       date,
    job              varchar2(255 char)
);

-- table index
create index employees_i1 on employees (department_id);




-- triggers
create or replace trigger employees_biu
    before insert or update
    on employees
    for each row
begin
    :new.email := lower(:new.email);
end employees_biu;
/


-- create views
create or replace view emp_v as
select
    departments.id           department_id,
    departments.name         department_name,
    departments.location     location,
    departments.country      country,
    employees.id             employee_id,
    employees.name           employee_name,
    employees.email          email,
    employees.cost_center    cost_center,
    employees.date_hired     date_hired,
    employees.job            job
from
    departments,
    employees
where
    employees.department_id(+) = departments.id/

-- load data

insert into departments (
    id,
    name,
    location,
    country
) values (
    1,
    'Delivery',
    'Garukme',
    'IL'
);
insert into departments (
    id,
    name,
    location,
    country
) values (
    2,
    'Manufacturing',
    'Covdiiku',
    'MH'
);
insert into departments (
    id,
    name,
    location,
    country
) values (
    3,
    'Sales',
    'Imaerosed',
    'VU'
);
insert into departments (
    id,
    name,
    location,
    country
) values (
    4,
    'Manufacturing',
    'Cugewpap',
    'CR'
);

commit;

insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    1,
    4,
    'Elnora Payne',
    'ketbeun@sim.gf',
    84,
    sysdate-86,
    'Analyst'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    2,
    1,
    'Katie Anderson',
    'binotse@wo.ec',
    78,
    sysdate-9,
    'Architect'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    3,
    3,
    'Myrtie Maldonado',
    'zovba@uf.kz',
    7,
    sysdate-87,
    'Salesman'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    4,
    1,
    'Carrie Carlson',
    'pihaw@zamuneb.ws',
    12,
    sysdate-77,
    'Manager'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    5,
    3,
    'Lucas Larson',
    'pok@zavig.cy',
    48,
    sysdate-79,
    'Consultant'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    6,
    2,
    'Leo Vargas',
    'pes@duche.qa',
    58,
    sysdate-75,
    'Engineer'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    7,
    3,
    'Verna Greene',
    'lepfo@kozhonmi.bo',
    68,
    sysdate-62,
    'Architect'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    8,
    1,
    'Walter Hodges',
    'zi@ti.gov',
    82,
    sysdate-17,
    'Consultant'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    9,
    3,
    'Franklin Nunez',
    'ewuwip@redfecuh.sk',
    68,
    sysdate-95,
    'Consultant'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    10,
    4,
    'Seth Tran',
    'ofosid@daej.mp',
    78,
    sysdate-25,
    'Engineer'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    11,
    1,
    'Della Page',
    'toosku@bakibhi.cf',
    88,
    sysdate-73,
    'Manager'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    12,
    2,
    'Nicholas Harrison',
    'hate@fu.ve',
    25,
    sysdate-29,
    'Architect'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    13,
    2,
    'Walter Lane',
    'laum@gu.de',
    10,
    sysdate-44,
    'Manager'
);
insert into employees (
    id,
    department_id,
    name,
    email,
    cost_center,
    date_hired,
    job
) values (
    14,
    3,
    'Edgar Little',
    'nulcun@gelet.cr',
    72,
    sysdate-38,
    'Salesman'
);

commit;


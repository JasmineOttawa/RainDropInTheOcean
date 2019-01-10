---
layout: post
title: Oracle PL/SQL - WHERE CURRENT OF   
category: Oracle 
tags: [Oracle]
---

Here is an example of using WHERE CURRENT OF to delete duplicate rows  

identify all the duplicate rows:   
```
	cursor books_to_fix is 
    select /*+ parallel(book, 32) */ book_name, book.cat_id, count(*)
    from book
    inner join cat on book.book_cat_id = cat.cat_id and cat.status='A'
    where book.status!='D'
    group by book_name, book.cat_id
    having count(*)>1;
```
select all the rows for a specific book:   
```
	cursor one_book_data(book_name_in  in book.book_name%TYPE) is
    select book_id, lastmod, status
    from book 
    where book_name = book_name_in and status!='D'
    order by lastmod desc for update of status;
 
    one_book_rec one_book_data%ROWTYPE;
    current_book_being_fixed book.book_name%TYPE;  
```
actual delete  
``` 
    for rec in books_to_fix loop
      dbms_output.put_line(chr(10)||'Fixing book with book_name '||rec.book_name||':');
      open one_book_data(rec.book_name);
      loop
        fetch one_book_data into one_book_rec;
        if one_user_data%rowcount = 1 then
        -- keep first row, which has latest lastmod
           dbms_output.put_line('......');
        else 
           -- generating undo 
           update book set status = 'D' where current of one_book_data;  
        end if;
      end loop;
      close one_book_data;
    end loop;
```

## Q & A   

### Q1 -  why we need "where current of"? without this one, it could be replaced by:  update book set status='D' where book_id = one_book_rec.book_id;   
    where current of removes the need of a primary/unique key when updating.   
  
### Q2 -  what does FOR UPDATE do?   
    when we use rowid as the unique key to identify a row, it may changes by operations like :   
    alter table ... shrink space; moving tablespaces or big DMLs.   
    FOR UPDATE locks the row, it ensures ROWID does not change until you finished your transaction.   
    
### Q3 -  difference between FOR UPDATE & FOR UPDATE OF ... 

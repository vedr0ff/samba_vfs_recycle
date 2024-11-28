# samba_vfs_recycle
Исследовать реализацию удаления AD-объектов через корзину (AD Recycle Bin) для возможности их восстановления.

1. Как реализовано удаление в корзину?  
   Реализация удаления в корзину:

   - вызовом функции [recycle_unlinkat](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L756) из таблицы операций в котором в свою очередь если не указан флаг  [AT_REMOVEDIR](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L740) (удаление пустого каталога) вызывается функция [recycle_unlink_internal](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L746) в которой реализован функционал перемещения в корзину.
   - функция [recycle_unlink_internal](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L473) начинается с получения параметров настройки модуля [vfs_recycle](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L492) и следующих шагов
     - далее следует [проверка](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L499) наличия пути к каталогу, в который следует переместить удаленные файлы
     - далее [получение](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L509) полного пути к перемещаемому файлу
     - далее [проверка](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L519) на попытку удалить саму корзину
     - далее [получение](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L529) размера файла
     - далее [проверка](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L547) размера на превышение количества байтов, указанное в параметре maxsize настройки модуля, если размер файла больше то он не будет помещены в репозиторий.
     - далее [проверка](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L557) размера на минимальное количества байтов, указанное в параметре minsize настройки модуля, если размер файла меньше то он не будет помещены в репозиторий.
     - далее [извлечение](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L584) пути и имени файла
     - далее [проверка](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L606) что файл не в списке файлов, которые при удалении не следует помещать в репозиторий, а удалять обычным способом.
     - далее [проверка](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L606) что путь к файлу не в списке каталогов, файлы которых при удалении не следует помещать в репозиторий, а удалять обычным способом.
     - далее [проверка](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L615) следует ли сохранять структуру каталогов или файлы в удаляемом каталоге следует хранить отдельно в репозитории, если ключ сохранения структуры установлен то [изменяем](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L616) путь к перемещению
     - далее [проверка](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L627) существует ли директория для перемещения, если нет то [создаём](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L632)
     - далее [формирование](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L648) полного пути из имени файла и пути до новой директории для перемещения
     - далее [создание](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L656) smb fname с полным путём перемещения файла
     - далее [проверка](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L675) существует ли старая версия файла и [перезаписывать](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L677) ли в корзине старую версию перемещаемого файла
     - перемещение файла в корзину в рамках одной файловой системы нужно [изменить](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L701) путь к этому файлу и [записать](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L706)
     - далее если должна обновляться дата доступа к файлу при его перемещении в репозитории - [изменение](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L726) даты доступа к перемещаемому файл
     - завершение работы функции
2. Это опциональный динамический модуль или базовая функциональность «ядра» системы?  
   Это опциональный динамический модуль

   - его инициализация осуществляется в [NTSTATUS vfs_recycle_init(TALLOC_CTX *ctx)](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L760)  вызываемая во время
     загрузки модуля для регистрации реализованных операций. Функция инициализации используется для
     регистрации модуля во время выполнения Samba. Функция инициализации передает три параметра в функцию регистрации VFS -  [smb_register_vfs()](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L762).
     - номер версии интерфейса, как константа [SMB_VFS_INTERFACE_VERSION](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L762) ,
     - [имя модуля](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L762), под которым ядро Samba его узнает, и
     - [таблица операций](https://gitlab.com/samba-team/samba/-/blob/master/source3/modules/vfs_recycle.c?ref_type=heads#L763)
3. По утверждению разработчиков из Samba Team, данная функциональность реализована,
   но «ломается» при репликации нескольких контроллеров домена. Возможные причины не проанализированы.

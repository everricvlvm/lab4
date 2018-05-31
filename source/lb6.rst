.. lab4 documentation master file, created by
   sphinx-quickstart on Wed May 23 22:09:07 2018.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Работа №6. Сообщения. Разделяемая память. Семафоры.
===================================================

Цели работы
-----------
Изучение системных вызовов для обмена данными между процессами.

Содержание работы
-----------------

1) Написать программу для обмена текстовыми сообщениями между процессами, с использованием механизма разделяемой памяти. Обеспечить синхронизацию обмена с помощью механизма семафоров.
2) Написать Makefile, обеспечивающий трансляцию, установку, очистку и удаление программы.

Ход работы
----------

Dockerfile
~~~~~~~~~~

Фаил сборки проекта в докер-образ:

  .. code-block:: dockerfile
   
      FROM alpine
      RUN apk update && apk upgrade && apk add nano make gcc build-base
      RUN mkdir work && cd work
      COPY lab6c.c /work
      COPY lab6s.c /work
      COPY Makefile /work
      COPY script.sh /work
      RUN cd /work && make build
      RUN cd /work && make install
      RUN cd /work && make clean
      CMD sh /work/script.sh


Shell-script
~~~~~~~~~~~~

  .. code-block:: bash
     
      cd /work
      make build
      make install
      make clean
      lab6s &
      lab6c & lab6c& lab6c& lab6c&

Исходный код сервера - lab5s.c
~~~~~~~~~~~~~~~~~~~~~

  .. code-block:: c
     
     #include <stdio.h>
     #include <unistd.h>
     #include <string.h>
     #include <sys/ipc.h>
     #include <sys/sem.h>
     #include <sys/shm.h>
     
     
     #define FTOKSEMPATH "/tmp/lab6sem" /*Путь к файлу, передаваемому ftok для набора семафоров */
     #define FTOKSHMPATH "/tmp/lab6shm" /*Путь к файлу, передаваемому ftok для разделяемого сегмента */
     #define FTOKID 1                   /*Идентификатор, передаваемый ftok */

     #define NUMSEMS 2                  /* Число семафоров в наборе */
     #define SIZEOFSHMSEG 512           /* Размер сегмента разделяемой памяти */
  
     #define NUMMSG 12                  /* Число принимаемых сообщений */
		  
		  
		  int main(void)
		  {
		    int semid, shmid;
		    key_t semkey, shmkey;
		    void *shm_address;
		    struct sembuf operations[2];
		    struct shmid_ds shmid_struct;
		    
		    /*Создание файлов для ftok*/
		    fclose(fopen(FTOKSEMPATH,"w"));
		    fclose(fopen(FTOKSHMPATH,"w"));

		    /*Создание IPC-ключей*/
		    semkey = ftok(FTOKSEMPATH,FTOKID);
		    if ( semkey == (key_t)-1 )
		    {
		      printf("Сервер: ошибка при выполнении %s\n","semkey = ftok(FTOKSEMPATH,FTOKID);");
		      return -1;
		    }
		    shmkey = ftok(FTOKSHMPATH,FTOKID);
		    if ( shmkey == (key_t)-1 )
		    {
		      printf("Сервер: ошибка при выполнении %s\n","shmkey = ftok(FTOKSHMPATH,FTOKID);");
		      return -1;
		    }
  
		    /*Создание набора семафоров с помощью IPC-ключей*/
		    semid = semget( semkey, NUMSEMS, 0666 | IPC_CREAT | IPC_EXCL);
		    if ( semid == -1 )
		    {
		      printf("Сервер: ошибка при выполнении %s\n","semid = semget( semkey, NUMSEMS, 0666 | IPC_CREAT | IPC_EXCL);");
		      return -1;
		    }
  
		    /* Произведём 2 семафора:
		    Истина в 0 семафоре - область разделяемой памяти используется.
		    Истина в 1 семафоре - область разделяемой памяти изменена клиентом.*/
  
		    /*Инициализация семафоров*/
		    if(semctl( semid, 0, SETVAL, 0) == -1)
		    {
		      printf("Сервер: ошибка инициализации 0 семафора: %s\n","semctl( semid, 0, SETVAL, 0) == -1.");
		      return -1;
		    }
  
		    if(semctl( semid, 1, SETVAL, 0) == -1)
		    {
		      printf("Сервер: ошибка инициализации 1 семафора: %s\n","semctl( semid, 1, SETVAL, 0) == -1.");
		      return -1;
		    }
  
		    /*Создание сегмента разделяемой памяти*/
		    shmid = shmget(shmkey, SIZEOFSHMSEG, 0666 | IPC_CREAT | IPC_EXCL);
		    if (shmid == -1)
		    {
		      printf("Сервер: ошибка при выполнении %s\n","shmid = shmget(shmkey, SIZEOFSHMSEG, 0666 | IPC_CREAT | IPC_EXCL);");
		      return -1;
		    }
  
		    /*Прикрепление сегмента разделяемой памяти, получение адреса*/
		    shm_address = shmat(shmid, NULL, 0);
		    if ( shm_address==NULL )
		    {
		      printf("Сервер: ошибка при выполнении %s\n","shm_address = shmat(shmid, NULL, 0);");
		      return -1;
		    }
		    printf("Сервер готов принимать сообщения от клиентов. Сервер настроен на прием %d сообщений.\n\n", NUMMSG);
  
		    /*Цикл обработки сообщений. Выполняется NUMMSG раз*/
		    for (int i = 0; i < NUMMSG; i++)
		    {
		      /* Сервер ожидает появления Истину на втором семафоре (сегмент разделяемой памяти изменен клиентом), затем выставляет 1 на первом семафоре (сегмент занят) */
		      operations[0].sem_num = 1;
		      operations[0].sem_op = -1;
		      operations[0].sem_flg = 0;
		      
		      operations[1].sem_num = 0;
		      operations[1].sem_op =  1;
		      operations[1].sem_flg = IPC_NOWAIT;
		      
		      if (semop( semid, operations, 2 ) == -1)
		      {
		        printf("Сервер: ошибка при выполнении %s\n","semop( semid, operations, 2 ) == -1.");
		      }
		      
		      /*Обработать сообщение, полученное от клиента*/
		      printf(">> #%s#\n", (char *) shm_address);
		      
		      /*Установить первый семафор в 0 (сегмент свободен)*/
		      operations[0].sem_num = 0;
		      operations[0].sem_op  = -1;
		      operations[0].sem_flg = IPC_NOWAIT;
		      
		      if (semop( semid, operations, 1 ) == -1)
		      {
		        printf("Сервер: ошибка при выполнении %s\n","semop( semid, operations, 1 ) == -1.");
			return -1;
		      }
		      
		      } /* Конец цикла обработки сообщений. */

		      if (semctl( semid, 1, IPC_RMID ) == -1)
		      {
		        printf("Сервер: ошибка освобождения семафоров: %s\n","semctl( semid, 1, IPC_RMID ) == -1.");
			return -1;
		      }
		      
		      if (shmdt( shm_address ) == -1)
		      {
		        printf("Сервер: ошибка открепления сегмента разделяемой памяти: %s\n","shmdt(shm_address) == -1.");
			return -1;
		      }
  
		      if (shmctl( shmid, IPC_RMID, &shmid_struct ) == -1)
		      {
		        printf("Сервер: ошибка освобождения сегмента разделяемой памяти: %s\n","shmctl(shmid, IPC_RMID, &shmid_struct) == -1.");
			return -1;
		      }
  
		      /*Удаление файлов для ftok*/
		      unlink(FTOKSHMPATH);
		      unlink(FTOKSEMPATH);
		      return 0;
		    }



Исходный код сервера - lab5s.c
~~~~~~~~~~~~~~~~~~~~~

  .. code-block:: c

     #include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <sys/types.h>
    
#define FTOKSEMPATH "/tmp/lab6sem" /*Путь к файлу, передаваемому ftok для набора семафоров */
#define FTOKSHMPATH "/tmp/lab6shm" /*Путь к файлу, передаваемому ftok для разделяемого сегмента */
#define FTOKID 1                   /*Идентификатор, передаваемый ftok */
    
#define NUMSEMS 2                  /* Число семафоров в наборе */
#define SIZEOFSHMSEG 512           /* Размер сегмента разделяемой памяти */

#define NUMMSG 3                   /* Число передаваемых сообщений */

int main(void)
{
  struct sembuf operations[3];
  void         *shm_address;
  int semid, shmid;
  key_t semkey, shmkey;
  
  /*Создание IPC-ключей*/
  semkey = ftok(FTOKSEMPATH,FTOKID);
  if ( semkey == (key_t)-1 )
    {
      printf("Клиент: ошибка при выполнении %s\n","semkey = ftok(FTOKSEMPATH,FTOKID);");
      return -1;
    }
  shmkey = ftok(FTOKSHMPATH,FTOKID);
  if ( shmkey == (key_t)-1 )
    {
      printf("Клиент: ошибка при выполнении %s\n","shmkey = ftok(FTOKSHMPATH,FTOKID);");
      return -1;
    }
  
  /*Получение набора семафоров с помощью IPC-ключей*/
  semid = semget( semkey, NUMSEMS, 0666);
  if ( semid == -1 )
    {
      printf("Клиент: ошибка при выполнении %s\n","semid = semget( semkey, NUMSEMS, 0666);");
      return -1;
    }
  
  /*Получение сегмента разделяемой памяти*/
  shmid = shmget(shmkey, SIZEOFSHMSEG, 0666);
  if (shmid == -1)
    {
      printf("Клиент: ошибка при выполнении %s\n","shmid = shmget(shmkey, SIZEOFSHMSEG, 0666);");
      return -1;
    }
  
  /*Прикрепление сегмента разделяемой памяти, получение адреса*/
  shm_address = shmat(shmid, NULL, 0);
  if ( shm_address==NULL )
    {
      printf("Клиент: ошибка при выполнении %s\n","shm_address = shmat(shmid, NULL, 0);");
      return -1;
    }
  
  /*Цикл отправки сообщений. Выполняется NUMMSG раз*/
  for (int i = 0; i < NUMMSG; i++)
    {
      /* Клиент ожидает появления Лжи на 0 семафоре (сегмент разделяемой памяти свободен) и Лжи на 1 семафоре (сегмент обработан сервером) затем выставляет Истину на 0 семафоре (сегмент занят) */
      
      operations[0].sem_num = 0;
      operations[0].sem_op =  0;
      operations[0].sem_flg = 0;
      
      operations[1].sem_num = 1;
      operations[1].sem_op =  0;
      operations[1].sem_flg = 0;
      
      operations[2].sem_num = 0;
      operations[2].sem_op =  1;
      operations[2].sem_flg = 0;
      
      if (semop( semid, operations, 3 ) == -1)
	{
	  printf("Клиент: ошибка при выполнении %s\n","semop( semid, operations, 2 ) == -1.");
	  return -1;
	}
      
      snprintf( (char *) shm_address, SIZEOFSHMSEG, "(Само сообщение) pid=%d", getpid() );
      usleep(200);
      /* Установить 0 семафор в 0 (сегмент свободен)
         Установить 1 семафор в 1 (сегмент изменен).*/
      operations[0].sem_num = 0;
      operations[0].sem_op =  -1;
      operations[0].sem_flg = 0;
      
      operations[1].sem_num = 1;
      operations[1].sem_op =  1;
      operations[1].sem_flg = 0;
      if (semop( semid, operations, 2 ) == -1)
	{
	  printf("Клиент: ошибка при изменении семафоров: %s\n","semop( semid, operations, 2 ) == -1.");
	  return -1;
	}
    }  /* Конец цикла отправки сообщений */
  
  /*Открепление сегмента разделяемой памяти.*/
  if (shmdt(shm_address) == -1)
    {
      printf("Клиент: ошибка при откреплении сегмента разделяемой памяти: %s\n","shmdt(shm_address) == -1.");
      return -1;
    }
  
  return 0;
}



Исходный код - Makefile
~~~~~~~~~~~~~~~~~~~~~~~

Фаил для упровления программой:

  .. code-block:: makefile
  
      build:
	       gcc lab6s.c -o lab6s
	       gcc lab6c.c -o lab6c
      clean:
	       rm -f lab6s lab6s.o
	       rm -f lab6c lab6c.o
      install:
	       cp lab6c /bin/lab6c
	       cp lab6s /bin/lab6s
      uninstall:
	       rm -f /bin/lab6c
	       rm -f /bin/lab6s

	       
Содержимое вывода
~~~~~~~~~~~~~~~~~

Требуемый вывод программы:

  .. code-block:: bash
     / # Сервер готов принимать сообщения от клиентов. Сервер настроен на прием 12 сообщений.

     >> #(Само сообщение) pid=7#
     >> #(Само сообщение) pid=10#
     >> #(Само сообщение) pid=9#
     >> #(Само сообщение) pid=8#
     >> #(Само сообщение) pid=7#
     >> #(Само сообщение) pid=10#
     >> #(Само сообщение) pid=9#
     >> #(Само сообщение) pid=8#
     >> #(Само сообщение) pid=7#
     >> #(Само сообщение) pid=10#
     >> #(Само сообщение) pid=9#
     >> #(Само сообщение) pid=8#

      
Образ
-----

Подготовленный образ для работы с ним через hub.docker.io:

  .. code-block:: bash
   
      docker run -it everricvlvm/lab6
	   
Выводы
------

Мною были изучены средства удалённого документирования и было опробовано взаимодействие облачных служб. Также я ознакомился с механизмом разделения памяти и методами воздействия на неё.
	   
.. toctree::
   :maxdepth: 2
   :caption: Contents:



.. lab4 documentation master file, created by
   sphinx-quickstart on Wed May 23 22:09:07 2018.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Работа №5. Переменные окружения и файловые операции.
====================================================

Цели работы
-----------
Изучение системных вызовов для работы с файлами и переменными окружения.

Содержание работы
-----------------

1) Написать программу, сохраняющею значения аргументов командной строки и параметров окружающей среды в файл lab5.txt.
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
      COPY lab5.c /workCOPY lab5.c /work
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
      lab5 qwe asd zxc

Исходный код - lab5.c
~~~~~~~~~~~~~~~~~~~~~

  .. code-block:: c

      #include <stdio.h>
      #include <unistd.h>
      #include <fcntl.h>
      #include <sys/types.h>
      
      int main(int argc, char* argv[], char* envp[])
      {
	       int fd = open("lb5.txt", O_WRONLY | O_CREAT | O_TRUNC, 0777);
	       write(fd, &"###arguments###\n\n", 17);
	       printf("###arguments###\n\n");
	       for (int i = 1; i<argc; i++)
	       {
	         write(fd, argv[i], strlen(argv[i]));
		 write(fd, &"\n", 1);
		 printf("%s\n", argv[i]);
	       }

	       write(fd, &"###enviroment###\n\n", 18);
	       printf("###enviroment###\n\n");

	       for(char** temp = envp; *(temp) != NULL; temp++)
	       {
	         write(fd, *temp, strlen(*temp));
		 write(fd, &"\n", 1);
		 printf("%s\n", *temp);
	       }
  
	       close(fd);
	       return 0;
      }

Исходный код - Makefile
~~~~~~~~~~~~~~~~~~~~~~~

Фаил для упровления программой:

  .. code-block:: makefile
  
      build:
	       gcc lab5.c -o lab5
      clean:
	       rm -f lab5 lab5.o
      install:
	       cp lab5 /bin/lab5
      uninstall:
	       rm -f /bin/lab5

	       
Содержимое файла lab5.txt ("lab5 qwe asd zxc")
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Требуемый вывод программы:

  .. code-block:: bash
  
      ###arguments###
      
      qwe
      asd
      zxc

      ###enviroment###
      
      LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:c$
      XDG_MENU_PREFIX=gnome-
      LANG=en_US.UTF-8
      GDM_LANG=en_US.UTF-8
      DISPLAY=:1
      COLORTERM=truecolor
      USERNAME=root
      XDG_VTNR=2
      SSH_AUTH_SOCK=/run/user/0/keyring/ssh
      S_COLORS=auto
      XDG_SESSION_ID=3
      USER=root
      DESKTOP_SESSION=gnome
      GNOME_TERMINAL_SCREEN=/org/gnome/Terminal/screen/4f179ed8_45a7_4eb3_8742_484929$
      PWD=/root/labs
      HOME=/root
      SSH_AGENT_PID=1204
      QT_ACCESSIBILITY=1
      XDG_SESSION_TYPE=x11
      XDG_DATA_DIRS=/usr/share/gnome:/usr/local/share/:/usr/share/
      XDG_SESSION_DESKTOP=gnome
      GJS_DEBUG_OUTPUT=stderr
      TK_MODULES=gail:atk-bridge
      WINDOWPATH=2
      TERM=xterm-256color
      SHELL=/bin/bash
      VTE_VERSION=5200
      XDG_CURRENT_DESKTOP=GNOME
      GPG_AGENT_INFO=/run/user/0/gnupg/S.gpg-agent:0:1
      GNOME_TERMINAL_SERVICE=:1.67
      SHLVL=1
      XDG_SEAT=seat0
      GDMSESSION=gnome
      GNOME_DESKTOP_SESSION_ID=this-is-deprecated
      LOGNAME=root
      DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/0/bus
      XDG_RUNTIME_DIR=/run/user/0
      XAUTHORITY=/run/user/0/gdm/Xauthority
      PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
      GJS_DEBUG_TOPICS=JS ERROR;JS LOG
      SESSION_MANAGER=local/kali:@/tmp/.ICE-unix/1109,unix/kali:/tmp/.ICE-unix/1109
      _=./lab5

Образ
-----

Подготовленный образ для работы с ним через hub.docker.io:

  .. code-block:: bash
   
      docker run -it everricvlvm/lab5
	   
Выводы
------

Мною были изучены средства удалённого документирования и было опробовано взаимодействие облачных служб. Также я ознакомился с некоторыми системными вызовами, аргументами и переменными окружения программы.
	   
.. toctree::
   :maxdepth: 2
   :caption: Contents:



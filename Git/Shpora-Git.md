## Git

### Git - начало

Git - система контроля версий которая фиксирует изменения в файле или файлах с течением времени, чтобы можно было вернутся к определенным веряиям в будущем

#### Схема локальной системы контроля версий

-- Local PC --

Checkout        Version Database
    |                |
	|                |
	|                |
  Files -------- Version 1
                 Version 2
				 Version 3
		

#### Централизованные системы контроля версий

-- PC (A) --    Central VCS Server
      |          
	  |         Version Database
	  |                |
	Files ------------ |
                  Version 1
-- PC (B) --      Version 2
      |           Version 3
      |                |
      |                |
    Files ------------ | 	  
	
#### Распределенные системы контроля версий

         Server Computer
		 
		 Version Database
		        |
				|
		    Version 1
			Version 2
			Version 3
			    |
				|
	 |----------|----------|
-- PC (A) --           -- PC (B) --
     |                     |
     |                     |
    Files                Files
     |                     |
     |                     |
  Version Database ---- Version Database
     |                     |
     |                     |
  Version 1              Version 1
  Version 2              Version 2
  Version 3              Version 3
  
#### Снимки

Git рассматривает данные как серию снимков файловой системы
При изменении или сохранении своего проекта, Git делает снимок того, как выглядят ваши файлы в этот момент, и сохраняет ссылку на этот снимок

Version 1       Version 2        Version 3        Version 4       Version 5
    |               |                |                |               |
    |               |                |                |               |
  Files A           A1               A1               A2              A2
    |               |                |                |               |
    |               |                |                |               |
  Files B           B                B                B1              B2
    |               |                |                |               |
    |               |                |                |               |
  Files C 	        C1               C2               C2              C3
  
#### Три состояния

В Git есть три основных состояния, в которых могут находится ваши файлы : измененный, подготовленный и зафиксированный

- "Измененный" означает, что вы изменили файла
- "Подгтовленный" означает, что вы отметили изменный файла
- "Зафиксированный" означает, что данные надежно сохранены в вашей локальной базе

-- Working Directory --              -- Staging Area --            -- .git directory (Repository) --  
            | <--------------------------------|---------- Chekout the project --------|
			|								   |		    						   |
			|								   |       								   |
			|-------- Stage Fixes     -------->|									   |
			|								   |									   |
			|								   |									   |
			|								   |------------- Commite ---------------->|
			
#### Команды

- Просмотр всех своих настроек: ''' git config --list --show-origin '''
- Указываем имя пользователя и почту: ''' git config --global user.name "Test Test" '''
                                      ''' git config --global user.mail test@example.com '''
- Указываем какой редактор использовать: ''' git config --global core.editor "'C:/Progam Files/Notepad++/notepad++.exe' -multiInst -notabbar -nosession -noPlugin" '''
- Меняем ветку по умолчанию: ''' git config --global init.defaultBranch main '''
- Проверить настройки конфигурации: ''' git config --list '''
- Получаем помощь: ''' git help '''
- Доступные параметры: ''' git add -h '''
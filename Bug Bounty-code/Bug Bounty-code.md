#web #parameter_pollution #parameter #pollution #blackbox

### **Разведка**
Во-первых проводим исследование инфраструктуры, которая у нас имеется.
У нас есть сайт с формой регистрации:
![[Pasted image 20241107003809.png | 400]]

После регистрации мы видим сервис для форматирования заметок:
![[Pasted image 20241107004130.png | 500]]

Потыкав пару кнопок мы натыкаемся на текст, который мы само собой **"не должны были увидеть"**:
![[Pasted image 20241107004321.png]]

Раскодировав из **base64** мы видим какую-то заметку, которая должна нас подталкивать на умные мысли:
```bash
└─$ echo 'IEkgdXBkYXRlZCB0aGUgdXNlciBlbnRpdHkgdmlldy4NCiBtb2RlbCA9IHsNCiAnbmlja25hbWUnOiBuaWNrbmFtZSwNCiAncGFzc3dvcmQnOiBwYXNzd29yZCwNCiAnaXNfYWRtaW4nOiAnZmFsc2UnDQogfQ==' | base64 -d

 I updated the user entity view.
 model = {
 'nickname': nickname,
 'password': password,
 'is_admin': 'false'
 }
```

Немного пораскинув мозгами мы понимаем, что нам надо двигаться в сторону изменения учётных параметров.

### **Атака**
> Я проверил довольно много вариантов, включая **SQLi**, **XSS**, **SSTI**, но всё это детектилось моментально, а значит нас ждёт что-то другое. на всё это угадывание у меня ушло пару дней.

Перехватив запрос через burp я обнаружил, что сервер ломается только из-за одного символа - символа двойной кавычки **"**. Отсюда и будем отталкиваться

Исходя из полученной скрытой заметки делаем предположение, что сервер собирает из наших отправляемых параметров **JSON**-структуру. Но как?! А вот для этого надо попытаться подумать как автор таска. У меня получилось как-то так:

```python
from flask import Flask

# какой-то код ...
app = Flask(__name__)

@app.route('/edit', methods=['POST'])
def edit():
	nickname = request.form.get('nickname')
	password = request.form.get('password')
	structure = f"""
	{
		"nickname": "{nickname}"
		"password": "{password}"
	}
	"""
	# какой-то код валидации JSON, обработки ошибок, подключения к БД,
	# применения изменений и отправки response
    return render_template('index.html')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

> Зачем так сложно? Собственно у меня был такой же вопрос, и после решения таска у меня появилось искреннее желание заглянуть в глаза автору с вопросом:
> > НАХЕРА, А ГЛАВНОЕ - ЗАЧЕМ????
> Но таск есть таск... Надо решать

После визуализации мы видим наш вектор атаки. Как он будет выглядеть? Да примерно вот так:
```json
{
	"nickname":"new_nickname",
	"password":"new_password","is_admin":"true"
}
```
А наш пэйлоад будет соответственно:
```json
new_password","is_admin":"true
```
Это и есть та строчка, которую мы укажем вместо пароля.

Перехватываем **request**:
![[Pasted image 20241107012003.png]]

Меняем его и отправляем:
![[Pasted image 20241107012225.png]]

Заходим под новыми кредами и видим, что теперь мы - админы:
![[Pasted image 20241107012321.png]]

Осталось только найти флаг, который ждёт нас в скрытой заметке:
![[Pasted image 20241107012550.png]]


Артисти — компактна проєкція

from Artists as a
order by a.display_name
select {
  id: id(a),
  name: a.display_name,
  type: a.artist_type,
  country: a.country
}

<img width="913" height="916" alt="image" src="https://github.com/user-attachments/assets/6e845807-c8b2-4ecc-9b4b-b34ee02d56dd" />


Колекція + load
from Events as e
order by e.date
load e.venue as v
select {
  event: e.name,
  date:  e.date,
  venue: v.display_name || v.name,
  city:  v.city || (v.address && v.address.city)
}


<img width="891" height="820" alt="image" src="https://github.com/user-attachments/assets/c3a3dec7-2b5e-47fe-bf0d-3cba73e6129e" />

Якщо поле дати інше (наприклад, startDate)

from Events as e
order by e.startDate
load e.venue as v
select {
  event: e.name,
  date:  e.startDate,
  venue: v.display_name || v.name,
  city:  v.city || (v.address && v.address.city)
}

<img width="876" height="832" alt="image" src="https://github.com/user-attachments/assets/2f5b30b9-90a9-4169-ab42-5676ea2facfe" />



із індексом (якщо маєш власний Events/ByDate)

order by ставимо до load, а load — перед select.

from index 'Events/ByDate' as e
order by e.date
load e.venue as v
select {
  event: e.name,
  date:  e.date,
  venue: v.display_name || v.name,
  city:  v.city || (v.address && v.address.city)
}


<img width="878" height="801" alt="image" src="https://github.com/user-attachments/assets/e188f981-4301-43eb-bf99-63e74c3ac3b2" />




поле дати існує й має саме таке ім’я;

from Events as e
order by e.date
select { event: e.name, venueId: e.venue }


<img width="877" height="711" alt="image" src="https://github.com/user-attachments/assets/badff99e-9648-4f79-8845-e5d2e99faa41" />

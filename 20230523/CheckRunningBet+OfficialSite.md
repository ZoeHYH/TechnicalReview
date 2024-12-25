---
marp: true
theme: default
paginate: true
---

# <!--fit--> Technical Review

Zoe
2023/05/23

---

# <!--fit--> Check Runnning Bet

---

![bg right](https://media.giphy.com/media/xTiTnEeKtzw4zJyFsQ/giphy.gif)

### Running bet checking will bet trigger every **half an hour** in following steps

1. GCP Cloud Scheduler
2. GCP Pub/Sub
3. GCP Cloud Function
4. Funky API

---

## ![bg left](https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExYTk4MWFhMWE5YzNiYmQwZjMyYzUzMzA4ZWJjMzhhYjdlYzY5ZWI5ZiZlcD12MV9pbnRlcm5hbF9naWZzX2dpZklkJmN0PWc/5t4gqIASvGzpBf9iMm/giphy-downsized-large.gif)

### Depends on game's type, there are two running bet conditions:

1. Half an hour

2. One day

---

## ![bg right](https://media.giphy.com/media/uKCrc7PFxJ9Xyya7DR/giphy.gif)

### We pull out running bets from Spanner and check with:

1. Operator

#### ...and also, if it is an **transfer game**:

2. Game Provider

---

![bg left](https://media.giphy.com/media/heIX5HfWgEYlW/giphy.gif)

### If the operator already settled or canceled bets

# We publish events to alter status

---

![bg right](https://media.giphy.com/media/G5Joz9CogWauYfFtln/giphy.gif)

### Then, if the game provider return pending status

# We ignore the bets

---

![bg left](https://media.giphy.com/media/huyZxIJvtqVeRp7QcS/giphy.gif)

### Finally, for the rest

# We notify gp with Skype and ourselves with Slack

---

# <!--fit--> Funky Official Site

---

![bg right](https://media.giphy.com/media/3o6UBaSkBsXM4gjgqc/giphy.gif)

# Fetch page order and content separately

---

![bg left](https://media.giphy.com/media/TejmLnMKgnmPInMQjV/giphy.gif)

# Handling data at one place

---

![bg right](https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExMjNiMmJhZDdlYWQwN2YyZjU5NTlhZjc2YTgwZTNjMDgwYmVhMWI4ZiZlcD12MV9pbnRlcm5hbF9naWZzX2dpZklkJmN0PWc/l0JMrPWRQkTeg3jjO/giphy.gif)

# Change SSR to SSG to load faster

---

![bg contain](https://media.giphy.com/media/xUPGcg1IJEKGCI6r5e/giphy.gif)

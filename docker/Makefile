.SHELLFLAGS := -ec
SECRET_KEY ?= /etc/default/secret.key

bootstrap:
	test $${STORAGE_PATH?Missing environment variable}
	mkdir -p $(abspath $(STORAGE_PATH))
	ln -s $(abspath $(STORAGE_PATH)) $(abspath storage)
	mkdir -p storage/silo/data
	make decrypt-secret

start:
	docker compose up -d --wait

stop:
	docker compose down

update:
	make stop
	git pull --rebase --depth=2
	docker compose pull
	make decrypt-secret
	make start
	docker system prune -a -f

listen:
	docker compose logs -f

create-secret-key:
	mkdir -p storage/secret
	openssl rand -out $(SECRET_KEY) -hex 64

encrypt-secret:
	for file in storage/secret/*; do \
		gpg --batch --pinentry-mode loopback --passphrase-file $(SECRET_KEY) -c --cipher-algo AES256 -o secret/$$(basename "$$file").gpg "$$file"; \
	done

decrypt-secret:
	mkdir -p storage/secret/
	for file in secret/*.gpg; do \
		gpg --batch --yes --pinentry-mode loopback --passphrase-file $(SECRET_KEY) -o storage/secret/$$(basename "$$file" .gpg) -d "$$file"; \
	done

scan:
	ggshield secret scan repo .
# Include variables from the .envrc file
include .envrc

# ==================================================================================== #
# HELPERS
# ==================================================================================== #

## help: print help message
.PHONY: help
help:
	@echo "Usage:"
	@sed -n 's/^##//p' ${MAKEFILE_LIST} | column -t -s ':' |  sed -e 's/^/ /'

# create the new confirm target
.PHONY: confirm
confirm:
	@echo -n "Are you sure you want to continue? [y/N] " && read ans && [ $${ans:-N} = y ]

# ==================================================================================== #
# DEVELOPMENT
# ==================================================================================== #

## run/api: run the ./cmd/api application
.PHONY: run/api
run/api:
	go run ./cmd/api -db-dsn=${GREENLIGHT_DB_DSN} -jwt-secret=${JWT_SECRET}

## db/psql: connect to the postgres database
.PHONY: db/psql
db/psql:
	docker exec -it my-postgres psql -U greenlight -d greenlight

## db/migrations/new name=$1: create a new migration files
.PHONY: db/migrations/new
db/migrations/new:
	@echo "Creating migration files for ${name}..."
	migrate create -seq -ext=.sql -dir=./migrations ${name}

# include confirm as a prerequisite
## db/migrations/up: apply all up migrations
.PHONY: db/migrations/up
db/migrations/up: confirm
	@echo "Running up migrations..."
	migrate -path ./migrations -database ${GREENLIGHT_DB_DSN} up

# ==================================================================================== #
# QUALITY CONTROL
# ==================================================================================== #

## audit: tidy dependencies and format, vet and test all code
.PHONY: audit
audit: vendor
	@echo "Formatting code..."
	go fmt ./...
	@echo "Vetting code..."
	go vet ./...
	staticcheck ./...
	@echo "Running tests..."
	go test -race -vet=off ./...

## vendor: tidy and vendor dependencies
.PHONY: vendor
vendor:
	@echo "Tidying and vendoring dependencies..."
	go mod tidy
	go mod verify
	@echo "Vendoring dependencies..."
	go mod vendor

# ==================================================================================== #
# BUILD
# ==================================================================================== #

current_time=$(shell date --iso-8601=seconds)
git_description=$(shell git describe --always --dirty --tags --long)
# linker_flags = '-s -X main.buildTime=${current_time} -X main.version=${git_description}'
linker_flags='-s -X main.buildTime=${current_time}'

## build/api: build the ./cmd/api application
.PHONY: build/api
build/api:
	@echo "Building ./cmd/api..."
	go build -ldflags=${linker_flags} -o=./bin/api ./cmd/api
	GOOS=linux GOARCH=amd64 go build -ldflags=${linker_flags} -o=./bin/api-linux-amd64 ./cmd/api

# ==================================================================================== #
# PRODUCTION
# ==================================================================================== #

production_host_ip='192.168.100.191'
production_host_port='2222'
## production/connect: connect to the production server
.PHONY: production/connect
production/connect:
	@echo "Connecting to production server..."
	ssh greenlight@${production_host_ip}

## production/deploy/api: deploy the api to production
.PHONY: production/deploy/api
production/deploy/api:
	@echo "Deploying api to production..."
	rsync -rP --delete ./bin/linux_amd64/api ./migrations greenlight@${production_host_ip}:~
#	sudo lxc file push -r ./bin/linux_amd64/api ./migrations my-ubuntu/greenlight
	ssh -t greenlight@${production_host_ip} 'migrate -path ~/migrations -database $$GREENLIGHT_DB_DSN up'

## production/configure/api.service: configure the production systemd api.service file
.PHONY: production/configure/api.service
production/configure/api.service:
	rsync -P ./remote/production/api.service greenlight@${production_host_ip}:~
	ssh -t greenlight@${production_host_ip} '\
		sudo mv ~/api.service /etc/systemd/system/ && \
		sudo systemctl enable api && \
		sudo systemctl restart api \
	'

## production/configure/caddyfile: configure the production Caddyfile
.PHONY: production/configure/caddyfile
production/configure/caddyfile:
	rsync -P ./remote/production/Caddyfile greenlight@${production_host_ip}:~
	ssh -t greenlight@${production_host_ip} '\
		sudo mv ~/Caddyfile /etc/caddy/ && \
		sudo systemctl reload caddy \
	'
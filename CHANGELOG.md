# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [2.2.0] - 2022-11-20

### Changed

- `README.md` changes and formatting
- `CHANGELOG.md` formatting
- Default `mail_dkim_size` changed to 2048 according to [RFC6376](https://tools.ietf.org/html/rfc6376)

## [2.1.0] - 2022-02-11

### Added

- Variable `mail_amavis_config` to configure Amavis

### Changed

- Order of Docker instalation and config generation

## [2.0.0] - 2022-02-02

### Added

- `galaxy.yml` file to provide more metadata for Ansible Galaxy
- This `CHANGELOG.md` file

### Changed

- (*breaking*) Switch to new repository [docker-mailserver](https://github.com/docker-mailserver/docker-mailserver) [#7](https://github.com/hmlkao/ansible-docker-mailserver/issues/7)

### Fixed

- Fixed Ansible files according to `ansible-lint`

## [1.0.0] - 2020-04-24

### Added

- Initial version

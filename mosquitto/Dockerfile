FROM eclipse-mosquitto
COPY config/mosquitto.conf /mosquitto/config/mosquitto.conf
# Default command from base image; override in compose if needed
CMD ["mosquitto", "-c", "/mosquitto/config/mosquitto.conf"]

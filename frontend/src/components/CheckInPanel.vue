<template>
	<div class="flex flex-col bg-white rounded w-full py-6 px-4 border-none">
		<h2 class="text-lg font-bold text-gray-900">Hey, {{ employee?.data?.first_name }} 👋</h2>

		<template v-if="settings.data?.allow_employee_checkin_from_mobile_app">
			<div class="font-medium text-sm text-gray-500 mt-1.5" v-if="lastLog">
				<span>Last {{ lastLogType }} was at {{ formatTimestamp(lastLog.time) }}</span>
				<span class="whitespace-pre"> &middot; </span>
				<router-link :to="{ name: 'EmployeeCheckinListView' }" v-slot="{ navigate }">
					<span @click="navigate" class="underline">View List</span>
				</router-link>
			</div>
			<Button
				class="mt-4 mb-1 drop-shadow-sm py-5 text-base"
				id="open-checkin-modal"
				@click="handleEmployeeCheckin"
			>
				<template #prefix>
					<FeatherIcon
						:name="nextAction.action === 'IN' ? 'arrow-right-circle' : 'arrow-left-circle'"
						class="w-4"
					/>
				</template>
				{{ nextAction.label }}
			</Button>
		</template>

		<div v-else class="font-medium text-sm text-gray-500 mt-1.5">
			{{ dayjs().format("ddd, D MMMM, YYYY") }}
		</div>
	</div>

	<ion-modal
		v-if="settings.data?.allow_employee_checkin_from_mobile_app"
		ref="modal"
		trigger="open-checkin-modal"
		:initial-breakpoint="1"
		:breakpoints="[0, 1]"
	>
		<div class="h-120 w-full flex flex-col items-center justify-center gap-5 p-4 mb-5">
			<div class="flex flex-col gap-1.5 mt-2 items-center justify-center">
				<div class="font-bold text-xl">
					{{ dayjs(checkinTimestamp).format("hh:mm:ss a") }}
				</div>
				<div class="font-medium text-gray-500 text-sm">
					{{ dayjs().format("D MMM, YYYY") }}
				</div>
			</div>

			<template v-if="settings.data?.allow_geolocation_tracking">
				<span v-if="locationStatus" class="font-medium text-gray-500 text-sm">
					{{ locationStatus }}
				</span>

				<div class="rounded border-4 translate-z-0 block overflow-hidden w-full h-170">
					<iframe
						width="100%"
						height="170"
						frameborder="0"
						scrolling="no"
						marginheight="0"
						marginwidth="0"
						style="border: 0"
						:src="`https://maps.google.com/maps?q=${latitude},${longitude}&hl=en&z=15&amp;output=embed`"
					>
					</iframe>
				</div>
			</template>

			<!-- Add Video Element for Selfie -->
			<video ref="video" autoplay playsinline class="rounded border-4" style="width: 100%; height: auto;"></video>
			<canvas ref="canvas" style="display:none;"></canvas>
			<Button class="mt-4" @click="takeSelfie" style="background-color: green; color: white;">Take Selfie</Button>
			<img v-if="photo" :src="photo" alt="Selfie" class="rounded border-4 mt-4" />

			<Button variant="solid" class="w-full py-5 text-sm" @click="submitLog(nextAction.action)">
				Confirm {{ nextAction.label }}
			</Button>
		</div>
	</ion-modal>
</template>


<script setup>
import { createResource, createListResource, toast, FeatherIcon } from "frappe-ui"
import { computed, inject, ref, onMounted, onBeforeUnmount } from "vue"
import { IonModal, modalController } from "@ionic/vue"

import { formatTimestamp } from "@/utils/formatters"

const DOCTYPE = "Employee Checkin"

const socket = inject("$socket")
const employee = inject("$employee")
const dayjs = inject("$dayjs")
const checkinTimestamp = ref(null)
const latitude = ref(0)
const longitude = ref(0)
const locationStatus = ref("")
const photo = ref(null)  // State to hold the selfie

const settings = createResource({
	url: "hrms.api.get_hr_settings",
	auto: true,
})

const checkins = createListResource({
	doctype: DOCTYPE,
	fields: ["name", "employee", "employee_name", "log_type", "time", "device_id"],
	filters: {
		employee: employee.data.name,
	},
	orderBy: "time desc",
})
checkins.reload()

const lastLog = computed(() => {
	if (checkins.list.loading || !checkins.data) return {}
	return checkins.data[0]
})

const lastLogType = computed(() => {
	return lastLog?.value?.log_type === "IN" ? "check-in" : "check-out"
})

const nextAction = computed(() => {
	return lastLog?.value?.log_type === "IN"
		? { action: "OUT", label: "Check Out" }
		: { action: "IN", label: "Check In" }
})

function handleLocationSuccess(position) {
	latitude.value = position.coords.latitude
	longitude.value = position.coords.longitude

	locationStatus.value =`
		Latitude: ${Number(latitude.value).toFixed(5)}°,
		Longitude: ${Number(longitude.value).toFixed(5)}°
	`
}

function handleLocationError(error) {
	locationStatus.value = "Unable to retrieve your location"
	if (error) locationStatus.value += `: ERROR(${error.code}): ${error.message}`
}

const fetchLocation = () => {
	if (!navigator.geolocation) {
		locationStatus.value = "Geolocation is not supported by your current browser"
	} else {
		locationStatus.value = "Locating..."
		navigator.geolocation.getCurrentPosition(handleLocationSuccess, handleLocationError)
	}
}

const handleEmployeeCheckin = async () => {
    checkinTimestamp.value = dayjs().format("YYYY-MM-DD HH:mm:ss");

    if (settings.data?.allow_geolocation_tracking) {
        fetchLocation();
    }
	https://meet.google.com/dgn-tszc-vur
    // Start the camera when checking in
    await startCamera();
};

const startCamera = async () => {
    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        try {
            const stream = await navigator.mediaDevices.getUserMedia({ video: true });
            const video = document.querySelector('video');
            video.srcObject = stream;
            video.play();

            // Debugging: Check if the stream is active
            console.log("Camera stream started:", stream.active);
        } catch (error) {
            console.error("Camera access error: ", error);
            alert('Unable to access the camera. Please check your browser settings.');
        }
    } else {
        alert("Camera is not supported by this browser.");
    }
};

const takeSelfie = () => {
    console.log("Taking selfie...");
    const video = document.querySelector('video');
    const canvas = document.querySelector('canvas');

    if (!video || video.readyState !== 4) {
        toast({
            title: "Error",
            text: "Camera is not accessible. Please check your camera settings.",
            icon: "alert-circle",
            position: "bottom-center",
            iconClasses: "text-red-500",
        });
        return;
    }

    canvas.width = video.videoWidth;
    canvas.height = video.videoHeight;

    // Check if video is playing
    console.log("Video ready state:", video.readyState);

    const context = canvas.getContext('2d');
    context.drawImage(video, 0, 0, canvas.width, canvas.height);

    photo.value = canvas.toDataURL('image/png');

    // Debugging: Check if the photo was captured
    console.log("Selfie captured:", photo.value);
};

// Stop the camera stream
const stopCamera = () => {
    const video = document.querySelector('video');
    const stream = video.srcObject;
    const tracks = stream.getTracks();

    tracks.forEach(track => {
        track.stop();  // Stop each track (video/audio)
    });

    video.srcObject = null;  // Clear the video source
};

// Update `submitLog` to stop the camera after submission
const submitLog = (logType) => {
    // Check if selfie is captured
    if (!photo.value) {
        // Show error message if selfie is not captured
        toast({
            title: "Error",
            text: "Please take a selfie before proceeding with the check-in/check-out.",
            icon: "alert-circle",
            position: "bottom-center",
            iconClasses: "text-red-500",
        });
        return; // Stop further execution
    }

    const action = logType === "IN" ? "Check-in" : "Check-out";
    console.log("submitting log", logType);

    // Proceed with the check-in/check-out submission
    checkins.insert.submit(
        {
            employee: employee.data.name,
            log_type: logType,
            time: checkinTimestamp.value,
            latitude: latitude.value,
            longitude: longitude.value,
            image: photo.value, // Set selfie as image
        },
        {
            onSuccess() {
                modalController.dismiss();
                toast({
                    title: "Success",
                    text: `${action} successful!`,
                    icon: "check-circle",
                    position: "bottom-center",
                    iconClasses: "text-green-500",
                });

                // Stop the camera after successful check-in/check-out
                stopCamera();
            },
            onError() {
                toast({
                    title: "Error",
                    text: `${action} failed!`,
                    icon: "alert-circle",
                    position: "bottom-center",
                    iconClasses: "text-red-500",
                });

                // Stop the camera even if the submission failed
                stopCamera();
            },
        }
    );
};



onMounted(() => {
	socket.emit("doctype_subscribe", DOCTYPE)
	socket.on("list_update", (data) => {
		if (data.doctype == DOCTYPE) {
			checkins.reload()
		}
	})
})

onBeforeUnmount(() => {
	socket.emit("doctype_unsubscribe", DOCTYPE)
	socket.off("list_update")
})
</script>

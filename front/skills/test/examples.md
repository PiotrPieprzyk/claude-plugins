# Examples of the shared files

Generic shapes taken from a working project (React + context-injected dependencies +
React Router). Rename to the project's domain; keep the structure.

## `<Domain>/shared/mocks.ts`

```ts
import {vi} from 'vitest';
import type {Router} from '@/domain/shared/service/Router';
import type {RoomAPI} from '@/domain/shared/api/RoomAPI';
import type {RoomEventsHandlers, RoomEventsSocket} from '@/domain/shared/service/RoomEventsSocket';

export function createMockRouter(): Router {
    return {push: vi.fn(), replace: vi.fn()};
}

export function createMockRoomAPI(overrides: Partial<RoomAPI> = {}): RoomAPI {
    return {
        getRoom: vi.fn(),
        updateRoom: vi.fn(),
        deleteRoom: vi.fn(),
        // ...every method of the interface
        ...overrides,
    };
}

// Event source: capture handlers so the test can emit, and signal "connected" like the real one.
export function createMockRoomEventsSocket() {
    let handlers: RoomEventsHandlers = {};
    const roomEventsSocket: RoomEventsSocket = {
        subscribe: vi.fn((h: RoomEventsHandlers) => { handlers = h; h.onConnect?.(); return vi.fn(); }),
        subscribeToRoom: vi.fn((_id: string, h: RoomEventsHandlers) => { handlers = h; h.onConnect?.(); return vi.fn(); }),
    };
    return {roomEventsSocket, getHandlers: () => handlers};
}
```

## `<Domain>/shared/builders.ts`

```ts
import type {RoomAPIResponseRaw} from '@/domain/shared/api/RoomAPI';

// Ids must pass the app's own validation (here: GUIDs parsed from the URL).
export const DEFAULT_ROOM_ID = crypto.randomUUID();

export function buildRoom(overrides: Partial<RoomAPIResponseRaw> = {}): RoomAPIResponseRaw {
    return {
        id: DEFAULT_ROOM_ID,
        name: 'ROOM_NULL_PTR',
        hostId: 'host-1',
        users: [{id: 'host-1', name: 'HostPlayer'}],
        status: 'WAITING_FOR_PLAYERS',
        ...overrides,
    };
}
```

## `<Domain>/shared/render.ts`

Providers in the same order as the composition root; router at the story's route so
params resolve; component imported by its public path.

```ts
import {createElement as h} from 'react';
import {render} from '@testing-library/react';
import {MemoryRouter, Route, Routes} from 'react-router';
import {RoomRenameForm} from '@/app/Room/RoomRenameForm';
import {RouterProvider} from '@/domain/shared/context/RouterContext';
import {RoomAPIProvider} from '@/domain/shared/context/RoomAPIContext';
import {DEFAULT_ROOM_ID} from '@docs/stories/Room/shared/builders';

export function renderRoomRenameForm(roomAPI: RoomAPI, router: Router, roomId = DEFAULT_ROOM_ID) {
    return render(
        h(MemoryRouter, {initialEntries: [`/rooms/${roomId}/rename`]},
            h(Routes, {},
                h(Route, {path: '/rooms/:roomId/rename', element:
                    h(RouterProvider, {router},
                        h(RoomAPIProvider, {roomAPI}, h(RoomRenameForm)))}))));
}
```

(`render.ts` is `.ts`, so it uses `createElement`; use `.tsx` and JSX if the project
prefers.)

## `<Domain>/shared/get/roomRenameForm.ts`

```ts
// RoomRenameForm (data-testid="roomRenameForm")
import {screen, within} from '@testing-library/react';

export function roomRenameForm() { return within(screen.getByTestId('roomRenameForm')); }
export function getRoomNameTextField() { return within(roomRenameForm().getByTestId('roomNameTextField')); }
export function getRoomNameInput() { return getRoomNameTextField().getByTestId('input'); }
export function getRoomNameErrorMessage() { return getRoomNameTextField().getByTestId('errorMessage'); }
export function getSaveRename() { return roomRenameForm().getByTestId('saveRename'); }
export function getCancel() { return roomRenameForm().getByTestId('cancel'); }
```

## A test with a loading state, validation, server error and cancel

```ts
describe('Host can rename the room', () => {
    it('WHEN the host types roomName "ValidName", saveRename SHOULD be clickable and when clicked SHOULD show loading state until redirected to #/rooms/{roomId}', async () => {
        const user = userEvent.setup();
        const router = createMockRouter();
        let resolveUpdate: (v: unknown) => void = () => {};
        const roomAPI = createMockRoomAPI({
            getRoom: vi.fn(async () => ok(buildRoom({name: 'ROOM_NULL_PTR'}))),
            updateRoom: vi.fn(() => new Promise((r) => { resolveUpdate = r; })), // pending on purpose
        });

        renderRoomRenameForm(roomAPI, router);
        await waitFor(() => expect(getRoomNameInput()).toHaveValue('ROOM_NULL_PTR'));

        await user.clear(getRoomNameInput());
        await user.type(getRoomNameInput(), 'ValidName');
        expect(getSaveRename()).toBeEnabled();

        await user.click(getSaveRename());
        expect(getSaveRename()).toHaveAttribute('aria-busy', 'true');
        expect(getSaveRename()).toBeDisabled();

        resolveUpdate(ok(buildRoom({name: 'ValidName'})));
        await waitFor(() => expect(router.push).toHaveBeenCalledWith(`#/rooms/${DEFAULT_ROOM_ID}`));
    });

    it('WHEN the host types roomName "Sh" saveRename SHOULD NOT be clickable and errorMessage should be visible', async () => {
        // ...render, wait, type 'Sh'
        expect(getRoomNameErrorMessage()).toHaveTextContent('ERR: ROOM_NAME_TOO_SHORT — (MIN 3 CHARS)');
        expect(getRoomNameErrorMessage()).toHaveStyle({color: DESIGN_COLORS.errorRed});
        expect(getSaveRename()).toBeDisabled();
    });

    it('WHEN the host submits a taken roomName SHOULD show errorMessage and SHOULD NOT navigate away', async () => {
        // updateRoom: vi.fn(async () => apiError('Room name already taken', 400))
        await waitFor(() => expect(getRoomNameErrorMessage()).toBeVisible());
        expect(getRoomNameErrorMessage()).toHaveTextContent('ERR: ROOM_NAME_TAKEN — TRY ANOTHER');
        expect(router.push).not.toHaveBeenCalled();
    });

    it('WHEN the host clicks cancel SHOULD be redirected to #/rooms/{roomId} without renaming', async () => {
        await user.click(getCancel());
        expect(router.push).toHaveBeenCalledWith(`#/rooms/${DEFAULT_ROOM_ID}`);
        expect(roomAPI.updateRoom).not.toHaveBeenCalled();
    });
});
```

## A realtime test

```ts
it('WHEN a roomDeleted event is received for a listed room SHOULD remove that roomListItem', async () => {
    const {roomEventsSocket, getHandlers} = createMockRoomEventsSocket();
    const roomAPI = createMockRoomAPI({getRooms: vi.fn(async () => ok(buildRoomList([buildRoom()])))});

    renderRoomList(roomAPI, createMockRouter(), roomEventsSocket);
    await waitFor(() => expect(getRoomListItems()).toHaveLength(1));

    act(() => getHandlers().onRoomDeleted?.(DEFAULT_ROOM_ID));

    await waitFor(() => expect(queryRoomListItems()).toHaveLength(0));
});
```
